# Tutorial: Migrating from CasaOS to ZimaOS on Proxmox

## Starting situation

- A **CasaOS** VM (Ubuntu under the hood) with one large disk (~1.3TB) holding all the data in `/DATA`
- A new **ZimaOS** VM (based on Buildroot, no `apt` or `lvm2`)
- The CasaOS data disk uses **LVM** internally (`ubuntu-vg/ubuntu-lv`)
- Goal: move the data to ZimaOS without copying 800GB over the network

---

## Step 1 — Identify the CasaOS disk from Proxmox

From the **Proxmox node shell**:

```bash
qm config <CASAOS_VM_ID>
```

Identify the large disk (e.g. `scsi1: local-lvm:vm-100-disk-3`).

---

## Step 2 — Expose the disk's partitions with kpartx

```bash
apt install kpartx -y

# View the disk's partition table
fdisk -l /dev/pve/vm-<ID>-disk-<N>

# Expose the partitions as devices
kpartx -av /dev/pve/vm-<ID>-disk-<N>
```

This creates devices like:
```
/dev/mapper/pve-vm--<ID>--disk--<N>p1
/dev/mapper/pve-vm--<ID>--disk--<N>p2
/dev/mapper/pve-vm--<ID>--disk--<N>p3
```

---

## Step 3 — Activate the CasaOS LVM from Proxmox

```bash
# Scan for the LVM inside the large partition (usually p3)
pvscan

# Activate the Volume Group (usually "ubuntu-vg")
vgchange -ay ubuntu-vg

# Mount the Logical Volume
mkdir -p /mnt/casaos-data
mount /dev/ubuntu-vg/ubuntu-lv /mnt/casaos-data

# Verify the data is there
ls /mnt/casaos-data/DATA
du -sh /mnt/casaos-data/DATA
```

---

## Step 4 — Identify the destination disk on ZimaOS

Check which disks are assigned to the ZimaOS VM:

```bash
qm config <ZIMAOS_VM_ID>
```

In this case there was a 1000GB disk formatted in **btrfs** (`vm-105-disk-2`) marked as `unused0`.

Add it to the ZimaOS VM:

```bash
qm set <ZIMAOS_VM_ID> -scsi2 local-lvm:vm-105-disk-2
```

---

## Step 5 — Mount the destination disk and copy the data

```bash
mkdir -p /mnt/nuevo-disco
mount /dev/pve/vm-<ID>-disk-2 /mnt/nuevo-disco

# Check available space
df -h /mnt/nuevo-disco

# Copy the data (the entire DATA folder)
rsync -av --progress /mnt/casaos-data/DATA/ /mnt/nuevo-disco/

# Also copy additional folders if there are any (e.g. /srv)
rsync -av --progress /mnt/casaos-data/srv/ /mnt/nuevo-disco/srv/
```

Monitor progress in another terminal:

```bash
watch -n 10 'df -h /mnt/nuevo-disco'
```

---

## Step 6 — Clean up the mounts when finished

Once everything is copied:

```bash
umount /mnt/nuevo-disco
umount /mnt/casaos-data
vgchange -an ubuntu-vg
kpartx -dv /dev/pve/vm-<ID>-disk-<N>
```

Verify everything is clean:

```bash
mount | grep mnt
lsblk | grep ubuntu
```

---

## Step 7 — Boot ZimaOS and configure storage

Start the ZimaOS VM:

```bash
qm start <ZIMAOS_VM_ID>
```

In the ZimaOS web interface, go to **Settings → Storage**. The disk should show up as `HDD-Storage` with the copied data.

Go to **Settings → Apps** and configure the paths:

| Path in ZimaOS | Folder |
|---|---|
| App data | `/media/HDD-Storage/AppData` |
| App image (docker) | `/media/HDD-Storage/docker` |
| User database | `/media/HDD-Storage/` |

> **Note:** If ZimaOS won't let you point to folders with existing content, temporarily rename the folders (add `-casaos` at the end), change the path, then rename them back to their original name after deleting the empty folders ZimaOS creates.

---

## Step 8 — Migrate the apps using CasaOS's docker-compose files

Copy the CasaOS compose folder to the ZimaOS disk (via SMB or another method):

```
/var/lib/casaos/apps/   →   directory with one docker-compose.yml per app
```

For each app, import the compose file in ZimaOS (**App Store → Import compose**) with these adjustments:

1. Remove the `x-casaos:` section at the end of the compose file (ZimaOS doesn't understand it)
2. Adjust the volume paths:
   - `/DATA/...` → `/DATA/...` (if ZimaOS has `/DATA` mounted on the disk)
   - `/srv/lsio/...` → `/media/HDD-Storage/srv/lsio/...` (custom paths)

### Example of a clean compose file for ZimaOS

```yaml
name: deluge
services:
    deluge:
        cpu_shares: 90
        container_name: deluge
        deploy:
            resources:
                limits:
                    memory: "4721737728"
                reservations:
                    memory: "268435456"
        environment:
            DELUGE_LOGLEVEL: error
            PGID: "1000"
            PUID: "1000"
            TZ: Etc/UTC
        hostname: deluge
        image: linuxserver/deluge:2.2.0
        network_mode: bridge
        ports:
            - target: 8112
              published: "8112"
              protocol: tcp
        restart: on-failure
        volumes:
            - type: bind
              source: /DATA/AppData/deluge/config
              target: /config
            - type: bind
              source: /DATA
              target: /DATA
            - type: bind
              source: /DATA/Downloads
              target: /downloads
networks:
    default:
        name: deluge_default
```

---

## Resulting folder structure on ZimaOS

```
/DATA/                          ← Large disk (HDD-Storage mounted)
├── AppData/                    ← App configurations
├── Downloads/
├── Documents/
├── Gallery/
├── Media/
│   ├── Movies/
│   └── TV Shows/
└── srv/
    └── lsio/                   ← linuxserver container configurations
        ├── jellyfin/
        ├── radarr/
        ├── prowlarr/
        └── duplicati/
```

---

## Important notes

- ZimaOS (Buildroot) **doesn't have `lvm2`**, which is why all the LVM activation work is done from Proxmox.
- The data disk ends up mounted on ZimaOS at `/media/HDD-Storage`, and is also accessible as `/DATA`.
- CasaOS compose files include metadata under `x-casaos:` that needs to be removed before importing into ZimaOS.
- linuxserver containers (`/srv/lsio/`) store their config outside of `/DATA`, so you need to use the full path when importing.
- For linuxserver containers, the `/config` volume must point to the **parent** folder that contains `config/`, `data/`, `cache/`, etc. — not directly to the `config/` subfolder.

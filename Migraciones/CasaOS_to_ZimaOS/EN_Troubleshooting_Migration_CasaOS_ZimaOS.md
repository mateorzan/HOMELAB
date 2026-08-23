# Troubleshooting: CasaOS → ZimaOS Migration on Proxmox

---

## 1. ZimaOS doesn't have `lvm2` — LVM can't be activated

**Problem:** ZimaOS is based on Buildroot and doesn't have `apt`, `vgscan`, `lvscan`, or any LVM tools.

**Solution:** Activate the LVM from the **Proxmox node shell**, which does have `lvm2`:

```bash
apt install kpartx -y
kpartx -av /dev/pve/vm-<ID>-disk-<N>
pvscan
vgchange -ay ubuntu-vg
mount /dev/ubuntu-vg/ubuntu-lv /mnt/casaos-data
```

---

## 2. `pvs` doesn't detect the CasaOS disk

**Problem:** Running `pvs` from Proxmox only shows the `pve` VG, not the CasaOS LVM.

**Cause:** The CasaOS LVM is inside a Proxmox LV (virtual disk), not on a direct physical device. LVM can't scan inside another LV.

**Solution:** Use `kpartx` to expose the internal partitions:

```bash
kpartx -av /dev/pve/vm-<ID>-disk-<N>
# Then scan the large partition (usually p3)
pvs /dev/mapper/pve-vm--<ID>--disk--<N>p3
```

---

## 3. `device is an LV` error when using pvs directly

**Problem:**
```
Cannot use /dev/pve/vm-105-disk-3: device is an LV
```

**Cause:** An attempt was made to scan the Proxmox LV directly without exposing its internal partitions.

**Solution:** Always use `kpartx` first to create the partition devices, then scan those devices.

---

## 4. Not enough space to copy the data on the same disk

**Problem:** The CasaOS disk had 810GB used and only 416GB free. Copying on the same disk wasn't possible.

**Solution:** Use an existing secondary disk in Proxmox (`vm-105-disk-2`, 1000GB in btrfs) that was showing as `unused0`. It was mounted from Proxmox and used as the `rsync` destination.

---

## 5. ZimaOS won't let you point to folders with existing content in Settings → Apps

**Problem:** When trying to change the `App data` path to a folder that already had data copied into it, ZimaOS showed the error `destination path is not empty` and wouldn't let you continue.

**Solution:** Temporarily rename the existing folders (add `-casaos`), change the path in ZimaOS (which now sees the folder as empty or nonexistent and accepts it), then from Proxmox delete the empty folders created by ZimaOS and rename the originals back to their correct name:

```bash
# From Proxmox with the disk mounted
rm -rf /mnt/nuevo-disco/AppData          # empty folder created by ZimaOS
mv /mnt/nuevo-disco/AppData-casaos /mnt/nuevo-disco/AppData
```

---

## 6. The AppData folder was accidentally deleted

**Problem:** When deleting the empty folder that ZimaOS had created, the `AppData` folder with the copied data (370GB) was mistakenly deleted along with it.

**Solution:** The original CasaOS disk was still intact in Proxmox as `unused1`. The LVM was reactivated and the data was copied again:

```bash
kpartx -av /dev/pve/vm-105-disk-3
vgchange -ay ubuntu-vg
mount /dev/ubuntu-vg/ubuntu-lv /mnt/casaos-data
mount /dev/pve/vm-105-disk-2 /mnt/nuevo-disco
rsync -av --progress /mnt/casaos-data/DATA/AppData/ /mnt/nuevo-disco/AppData/
```

---

## 7. `qm set --delete unusedX` physically deletes the disk

**⚠️ CRITICAL WARNING:** In Proxmox, the command `qm set <ID> --delete unusedX`, when the disk is in `unused` state, **physically removes the LV** from the pool, not just the reference.

**What NOT to do:**
```bash
qm set 105 --delete unused1   # ❌ PHYSICALLY DELETES THE DISK
```

**What you should do** if you just want to unassign the disk in order to move it to another VM:

First verify the disk has no active mappings:
```bash
kpartx -dv /dev/pve/vm-105-disk-3
vgchange -an ubuntu-vg
```

Then reassign it directly to the new VM without going through `--delete`:
```bash
qm set <NEW_VM_ID> -scsi0 local-lvm:vm-<ID>-disk-<N>
```

---

## 8. `dmsetup remove_all --force` left Proxmox inaccessible

**Problem:** Running `dmsetup remove_all --force` to clean up leftover LVM mappings also deactivated Proxmox's own LVs (`pve-root`, `pve-data`, etc.), leaving the web interface inaccessible.

**Solution:** Reboot the Proxmox node. On boot, Proxmox automatically reactivates its own `pve` VG.

```bash
reboot
```

> **Lesson:** Never use `dmsetup remove_all --force` on a production Proxmox node. To clean up mappings for a specific disk, always use `kpartx -dv` and `vgchange -an <vg_name>` selectively.

---

## 9. ctoz fails with a 404 error during online migration

**Problem:**
```
Step failed: Download and process source data - Failed to download files: Download failed, status code: 404
```

**Cause:** ctoz was trying to download the data from CasaOS but couldn't find the files at the expected path, because the data disk wasn't properly mounted on the restored CasaOS.

**Alternative solution:** Manually copy the docker-compose files from CasaOS to ZimaOS via SMB and import them one by one from the ZimaOS interface.

The CasaOS compose files are located at:
```
/var/lib/casaos/apps/<app_name>/docker-compose.yml
```

---

## 10. CasaOS compose files don't import into ZimaOS

**Problem:** When importing a CasaOS compose file into ZimaOS, the app either doesn't get created or fails silently.

**Causes:**
- The `x-casaos:` section at the end of the compose file isn't compatible with ZimaOS
- Volume paths point to `/DATA/...` or `/srv/...`, which may not exist on ZimaOS
- Some compose files have `x-casaos:` nested inside the service in addition to the global one

**Solution:** Clean up the compose file before importing:
1. Remove the entire `x-casaos:` section (both the global one and any nested inside services)
2. Adjust volume paths to the actual paths on ZimaOS
3. Remove empty `command: []` fields if they cause problems

---

## 11. Jellyfin starts as a fresh install, ignoring existing configuration

**Problem:** Jellyfin would install but show the initial setup wizard, without recognizing the existing data.

**Identified causes:**

**Cause A:** The container was installed from the ZimaOS store instead of importing the custom compose file. In that case it only mounted `/DATA/Media` but not `/config`.

Check with:
```bash
docker inspect <container_name> | grep -A5 "Binds\|Mounts"
```

**Cause B:** The `/config` volume path pointed to the wrong subfolder. linuxserver containers expect `/config` to be the **parent** folder that internally contains `config/`, `data/`, `cache/`, etc.

```yaml
# ❌ Incorrect
source: /media/HDD-Storage/srv/lsio/jellyfin/config
target: /config

# ✅ Correct
source: /media/HDD-Storage/srv/lsio/jellyfin
target: /config
```

**Solution:** Remove the container, fix the compose file, and reimport:

```bash
docker stop jellyfin
docker rm jellyfin
```

---

## 12. The `/srv` folder doesn't show up under `/DATA` on ZimaOS

**Problem:** The `/DATA/srv/lsio/...` paths didn't exist on ZimaOS even though the disk was mounted.

**Cause:** On ZimaOS, `/DATA` and `/media/HDD-Storage` are different mount points. The `srv` folder was copied to `/media/HDD-Storage/srv/`, but `/DATA` only shows the folders ZimaOS manages directly.

**Solution:** Use the full path `/media/HDD-Storage/srv/lsio/...` in the compose files, the same way it's done for `/media/HDD-Storage/AppData/...`.

---

## Summary of lessons learned

| Action | Risk | Safe alternative |
|---|---|---|
| `qm set --delete unusedX` | Physically deletes the disk if it's in `unused` state | Reassign it directly with `qm set <new_vm> -scsiX` |
| `dmsetup remove_all --force` | Can leave Proxmox inaccessible | Use `kpartx -dv` and `vgchange -an <vg>` selectively |
| Installing an app from the ZimaOS store instead of importing the compose file | The app won't use your existing configuration | Always import the original compose file with the correct paths |
| Deleting "empty" folders without checking | Can delete data if paths get confused | Always run `ls` before any `rm -rf` |

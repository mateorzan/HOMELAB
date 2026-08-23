🇬🇧 English | [🇪🇸 Español](README.es.md)

# CasaOS → ZimaOS Migration on Proxmox

Migrating a self-hosted stack from **CasaOS** to **ZimaOS** on Proxmox, moving ~800GB of data disk-to-disk instead of over the network, reactivating LVM from the Proxmox host (since ZimaOS/Buildroot has no `lvm2`), and porting the docker-compose apps across.

[![Watch the video](https://img.youtube.com/vi/VIDEO_ID/maxresdefault.jpg)](https://www.youtube.com/watch?v=VIDEO_ID)

## Contents

- 📘 [Tutorial](EN_Tutorial_Migration_CasaOS_ZimaOS.md) — step-by-step walkthrough of the full migration
- 🛠️ [Troubleshooting](EN_Troubleshooting_Migration_CasaOS_ZimaOS.md) — every issue hit along the way and how it was fixed

## Key lessons

| Action                                                                        | Risk                                                   | Safe alternative                                               |
| ----------------------------------------------------------------------------- | ------------------------------------------------------ | -------------------------------------------------------------- |
| `qm set --delete unusedX`                                                   | Physically deletes the disk if it's in`unused` state | Reassign it directly with`qm set <new_vm> -scsiX`            |
| `dmsetup remove_all --force`                                                | Can leave Proxmox inaccessible                         | Use`kpartx -dv` and `vgchange -an <vg>` selectively        |
| Installing an app from the ZimaOS store instead of importing the compose file | The app won't use your existing configuration          | Always import the original compose file with the correct paths |
| Deleting "empty" folders without checking                                     | Can delete data if paths get confused                  | Always run`ls` before any `rm -rf`                         |

## Stack involved

CasaOS (Ubuntu) · ZimaOS (Buildroot) · Proxmox · LVM · kpartx · rsync · Docker Compose · linuxserver.io containers (Jellyfin, Radarr, Prowlarr, Duplicati) · Deluge

---

Part of my [HOMELAB](../../) documentation.

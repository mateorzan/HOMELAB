[🇬🇧 English](README.md) | 🇪🇸 Español

# Migración CasaOS → ZimaOS en Proxmox

Migración de un stack self-hosted de **CasaOS** a **ZimaOS** en Proxmox, moviendo ~800GB de datos de disco a disco en lugar de por red, reactivando el LVM desde el host Proxmox (ya que ZimaOS/Buildroot no tiene `lvm2`), y migrando las apps con docker-compose.

[![Ver el vídeo](https://img.youtube.com/vi/VIDEO_ID/maxresdefault.jpg)](https://www.youtube.com/watch?v=VIDEO_ID)

## Contenido

- 📘 [Tutorial](ES_Tutorial_Migracion_CasaOS_ZimaOS.md) — guía paso a paso de toda la migración
- 🛠️ [Troubleshooting](ES_Troubleshooting_Migracion_CasaOS_ZimaOS.md) — todos los problemas encontrados y cómo se resolvieron

## Lecciones clave

| Acción                                                        | Riesgo                                                   | Alternativa segura                                            |
| -------------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------- |
| `qm set --delete unusedX`                                    | Borra el disco físicamente si está en estado`unused` | Reasignar directamente con`qm set <nueva_vm> -scsiX`        |
| `dmsetup remove_all --force`                                 | Puede dejar Proxmox inaccesible                          | Usar`kpartx -dv` y `vgchange -an <vg>` de forma selectiva |
| Instalar app desde tienda de ZimaOS en vez de importar compose | La app no usa tu configuración existente                | Siempre importar el compose original con las rutas correctas  |
| Borrar carpetas "vacías" sin verificar                        | Puede borrar datos si hay confusión entre rutas         | Siempre hacer`ls` antes de cualquier `rm -rf`             |

## Stack implicado

CasaOS (Ubuntu) · ZimaOS (Buildroot) · Proxmox · LVM · kpartx · rsync · Docker Compose · contenedores linuxserver.io (Jellyfin, Radarr, Prowlarr, Duplicati) · Deluge

---

Parte de mi documentación de [HOMELAB](../../).

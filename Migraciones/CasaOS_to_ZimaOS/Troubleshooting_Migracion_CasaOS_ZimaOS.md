# Troubleshooting: Migración CasaOS → ZimaOS en Proxmox

---

## 1. ZimaOS no tiene `lvm2` — no se puede activar el LVM

**Problema:** ZimaOS está basado en Buildroot y no tiene `apt`, `vgscan`, `lvscan` ni ninguna herramienta LVM.

**Solución:** Activar el LVM desde la **shell del nodo Proxmox**, que sí tiene `lvm2`:

```bash
apt install kpartx -y
kpartx -av /dev/pve/vm-<ID>-disk-<N>
pvscan
vgchange -ay ubuntu-vg
mount /dev/ubuntu-vg/ubuntu-lv /mnt/casaos-data
```

---

## 2. `pvs` no detecta el disco de CasaOS

**Problema:** Al ejecutar `pvs` desde Proxmox, solo aparece el VG `pve` y no el LVM de CasaOS.

**Causa:** El LVM de CasaOS está dentro de un LV de Proxmox (disco virtual), no en un dispositivo físico directo. LVM no puede escanear dentro de otro LV.

**Solución:** Usar `kpartx` para exponer las particiones internas:

```bash
kpartx -av /dev/pve/vm-<ID>-disk-<N>
# Luego escanear la partición grande (normalmente p3)
pvs /dev/mapper/pve-vm--<ID>--disk--<N>p3
```

---

## 3. Error `device is an LV` al usar pvs directamente

**Problema:**
```
Cannot use /dev/pve/vm-105-disk-3: device is an LV
```

**Causa:** Se intentó escanear el LV de Proxmox directamente sin exponer sus particiones internas.

**Solución:** Siempre usar `kpartx` primero para crear los dispositivos de partición, luego escanear sobre esos dispositivos.

---

## 4. No hay espacio suficiente para copiar los datos en el mismo disco

**Problema:** El disco de CasaOS tenía 810GB usados y solo 416GB libres. No era posible hacer la copia en el mismo disco.

**Solución:** Usar un disco secundario ya existente en Proxmox (`vm-105-disk-2`, 1000GB en btrfs) que estaba como `unused0`. Se montó desde Proxmox y se usó como destino del `rsync`.

---

## 5. ZimaOS no deja apuntar a carpetas con contenido existente en Settings → Apps

**Problema:** Al intentar cambiar la ruta de `App data` a una carpeta que ya tenía datos copiados, ZimaOS mostraba el error `destination path is not empty` y no permitía continuar.

**Solución:** Renombrar temporalmente las carpetas existentes (añadir `-casaos`), cambiar la ruta en ZimaOS (que ahora ve la carpeta vacía o inexistente y lo acepta), y luego desde Proxmox eliminar las carpetas vacías creadas por ZimaOS y renombrar las originales a su nombre correcto:

```bash
# Desde Proxmox con el disco montado
rm -rf /mnt/nuevo-disco/AppData          # carpeta vacía creada por ZimaOS
mv /mnt/nuevo-disco/AppData-casaos /mnt/nuevo-disco/AppData
```

---

## 6. Se borró accidentalmente la carpeta AppData

**Problema:** Al borrar la carpeta vacía que ZimaOS había creado, se borró también por error la carpeta `AppData` con los datos copiados (370GB).

**Solución:** El disco original de CasaOS seguía intacto en Proxmox como `unused1`. Se reactivó el LVM y se volvió a copiar:

```bash
kpartx -av /dev/pve/vm-105-disk-3
vgchange -ay ubuntu-vg
mount /dev/ubuntu-vg/ubuntu-lv /mnt/casaos-data
mount /dev/pve/vm-105-disk-2 /mnt/nuevo-disco
rsync -av --progress /mnt/casaos-data/DATA/AppData/ /mnt/nuevo-disco/AppData/
```

---

## 7. `qm set --delete unusedX` borra el disco físicamente

**⚠️ ADVERTENCIA CRÍTICA:** En Proxmox, el comando `qm set <ID> --delete unusedX` cuando el disco está en estado `unused` **elimina el LV físicamente** del pool, no solo la referencia.

**Lo que NO hay que hacer:**
```bash
qm set 105 --delete unused1   # ❌ BORRA EL DISCO FÍSICAMENTE
```

**Lo que hay que hacer** si solo quieres desasignar el disco para moverlo a otra VM:

Primero verificar que el disco no tiene mapeos activos:
```bash
kpartx -dv /dev/pve/vm-105-disk-3
vgchange -an ubuntu-vg
```

Y luego reasignarlo directamente a la nueva VM sin pasar por `--delete`:
```bash
qm set <ID_NUEVA_VM> -scsi0 local-lvm:vm-<ID>-disk-<N>
```

---

## 8. `dmsetup remove_all --force` dejó Proxmox inaccesible

**Problema:** Al ejecutar `dmsetup remove_all --force` para limpiar mapeos LVM residuales, se desactivaron también los LVs del propio Proxmox (`pve-root`, `pve-data`, etc.), dejando la interfaz web inaccesible.

**Solución:** Reiniciar el nodo Proxmox. Al arrancar, Proxmox reactiva automáticamente su propio VG `pve`.

```bash
reboot
```

> **Lección:** Nunca usar `dmsetup remove_all --force` en un nodo Proxmox en producción. Para limpiar mapeos de un disco concreto usar siempre `kpartx -dv` y `vgchange -an <vg_name>` de forma selectiva.

---

## 9. ctoz falla con error 404 al intentar migración online

**Problema:**
```
Step failed: Download and process source data - Failed to download files: Download failed, status code: 404
```

**Causa:** ctoz intentaba descargar los datos desde CasaOS pero no encontraba los archivos en la ruta esperada porque el disco de datos no estaba montado correctamente en CasaOS restaurada.

**Solución alternativa:** Copiar manualmente los docker-compose desde CasaOS a ZimaOS vía SMB e importarlos uno a uno desde la interfaz de ZimaOS.

Los compose de CasaOS se encuentran en:
```
/var/lib/casaos/apps/<nombre_app>/docker-compose.yml
```

---

## 10. Los compose de CasaOS no se importan en ZimaOS

**Problema:** Al importar un compose de CasaOS en ZimaOS, la app no se crea o falla silenciosamente.

**Causas:**
- La sección `x-casaos:` al final del compose no es compatible con ZimaOS
- Las rutas de volúmenes apuntan a `/DATA/...` o `/srv/...` que pueden no existir en ZimaOS
- Algunos compose tienen `x-casaos:` anidado dentro del servicio además del global

**Solución:** Limpiar el compose antes de importar:
1. Eliminar toda la sección `x-casaos:` (tanto la global como la anidada en servicios)
2. Ajustar rutas de volúmenes a las rutas reales en ZimaOS
3. Eliminar campos `command: []` vacíos si causan problemas

---

## 11. Jellyfin arranca como instalación nueva ignorando la configuración existente

**Problema:** Jellyfin se instalaba pero aparecía el asistente de configuración inicial, sin reconocer los datos existentes.

**Causas identificadas:**

**Causa A:** El contenedor fue instalado desde la tienda de ZimaOS en lugar de importar el compose personalizado. En ese caso solo montaba `/DATA/Media` pero no `/config`.

Verificar con:
```bash
docker inspect <nombre_contenedor> | grep -A5 "Binds\|Mounts"
```

**Causa B:** La ruta del volumen `/config` apuntaba a la subcarpeta equivocada. Los contenedores linuxserver esperan que `/config` sea la carpeta **padre** que contiene internamente `config/`, `data/`, `cache/`, etc.

```yaml
# ❌ Incorrecto
source: /media/HDD-Storage/srv/lsio/jellyfin/config
target: /config

# ✅ Correcto
source: /media/HDD-Storage/srv/lsio/jellyfin
target: /config
```

**Solución:** Eliminar el contenedor, corregir el compose y reimportar:

```bash
docker stop jellyfin
docker rm jellyfin
```

---

## 12. La carpeta `/srv` no aparece en `/DATA` de ZimaOS

**Problema:** Las rutas `/DATA/srv/lsio/...` no existían en ZimaOS aunque el disco estaba montado.

**Causa:** En ZimaOS, `/DATA` y `/media/HDD-Storage` son puntos de montaje diferentes. La carpeta `srv` fue copiada a `/media/HDD-Storage/srv/` pero `/DATA` solo mostraba las carpetas que ZimaOS gestiona directamente.

**Solución:** Usar la ruta completa `/media/HDD-Storage/srv/lsio/...` en los compose, igual que se hace con `/media/HDD-Storage/AppData/...`.

---

## Resumen de lecciones aprendidas

| Acción | Riesgo | Alternativa segura |
|---|---|---|
| `qm set --delete unusedX` | Borra el disco físicamente si está en estado `unused` | Reasignar directamente con `qm set <nueva_vm> -scsiX` |
| `dmsetup remove_all --force` | Puede dejar Proxmox inaccesible | Usar `kpartx -dv` y `vgchange -an <vg>` de forma selectiva |
| Instalar app desde tienda de ZimaOS en vez de importar compose | La app no usa tu configuración existente | Siempre importar el compose original con las rutas correctas |
| Borrar carpetas "vacías" sin verificar | Puede borrar datos si hay confusión entre rutas | Siempre hacer `ls` antes de cualquier `rm -rf` |

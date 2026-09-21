# Tutorial: Migración de CasaOS a ZimaOS en Proxmox

## Situación de partida

- VM con **CasaOS** (Ubuntu bajo el capó) con un disco grande (~1.3TB) con todos los datos en `/DATA`
- Nueva VM con **ZimaOS** (basado en Buildroot, sin `apt` ni `lvm2`)
- El disco de datos de CasaOS usa **LVM** internamente (`ubuntu-vg/ubuntu-lv`)
- Objetivo: mover los datos a ZimaOS sin copiar 800GB por red

---

## Paso 1 — Identificar el disco de CasaOS desde Proxmox

Desde la **shell del nodo Proxmox**:

```bash
qm config <ID_VM_CASAOS>
```

Identifica el disco grande (ej. `scsi1: local-lvm:vm-100-disk-3`).

---

## Paso 2 — Exponer las particiones del disco con kpartx

```bash
apt install kpartx -y

# Ver la tabla de particiones del disco
fdisk -l /dev/pve/vm-<ID>-disk-<N>

# Exponer las particiones como dispositivos
kpartx -av /dev/pve/vm-<ID>-disk-<N>
```

Esto crea dispositivos como:
```
/dev/mapper/pve-vm--<ID>--disk--<N>p1
/dev/mapper/pve-vm--<ID>--disk--<N>p2
/dev/mapper/pve-vm--<ID>--disk--<N>p3
```

---

## Paso 3 — Activar el LVM de CasaOS desde Proxmox

```bash
# Escanear el LVM dentro de la partición grande (normalmente p3)
pvscan

# Activar el Volume Group (normalmente "ubuntu-vg")
vgchange -ay ubuntu-vg

# Montar el Logical Volume
mkdir -p /mnt/casaos-data
mount /dev/ubuntu-vg/ubuntu-lv /mnt/casaos-data

# Verificar que los datos están ahí
ls /mnt/casaos-data/DATA
du -sh /mnt/casaos-data/DATA
```

---

## Paso 4 — Identificar el disco destino en ZimaOS

Verificar qué discos tiene asignados la VM de ZimaOS:

```bash
qm config <ID_VM_ZIMAOS>
```

En este caso había un disco de 1000GB formateado en **btrfs** (`vm-105-disk-2`) marcado como `unused0`.

Añadirlo a la VM de ZimaOS:

```bash
qm set <ID_VM_ZIMAOS> -scsi2 local-lvm:vm-105-disk-2
```

---

## Paso 5 — Montar el disco destino y copiar los datos

```bash
mkdir -p /mnt/nuevo-disco
mount /dev/pve/vm-<ID>-disk-2 /mnt/nuevo-disco

# Verificar espacio disponible
df -h /mnt/nuevo-disco

# Copiar los datos (carpeta DATA completa)
rsync -av --progress /mnt/casaos-data/DATA/ /mnt/nuevo-disco/

# Copiar también carpetas adicionales si las hay (ej. /srv)
rsync -av --progress /mnt/casaos-data/srv/ /mnt/nuevo-disco/srv/
```

Monitorizar el progreso en otra terminal:

```bash
watch -n 10 'df -h /mnt/nuevo-disco'
```

---

## Paso 6 — Limpiar los montajes al terminar

Una vez copiado todo:

```bash
umount /mnt/nuevo-disco
umount /mnt/casaos-data
vgchange -an ubuntu-vg
kpartx -dv /dev/pve/vm-<ID>-disk-<N>
```

Verificar que quedó limpio:

```bash
mount | grep mnt
lsblk | grep ubuntu
```

---

## Paso 7 — Arrancar ZimaOS y configurar el almacenamiento

Arrancar la VM de ZimaOS:

```bash
qm start <ID_VM_ZIMAOS>
```

En la interfaz web de ZimaOS ir a **Settings → Storage**. El disco debería aparecer como `HDD-Storage` con los datos copiados.

Ir a **Settings → Apps** y configurar las rutas:

| Ruta en ZimaOS | Carpeta |
|---|---|
| App data | `/media/HDD-Storage/AppData` |
| App image (docker) | `/media/HDD-Storage/docker` |
| User database | `/media/HDD-Storage/` |

> **Nota:** Si ZimaOS no permite apuntar a carpetas con contenido existente, renombrar temporalmente las carpetas (añadir `-casaos` al final), cambiar la ruta, y luego renombrarlas de nuevo a su nombre original eliminando las carpetas vacías que crea ZimaOS.

---

## Paso 8 — Migrar las apps con los docker-compose de CasaOS

Copiar la carpeta de compose de CasaOS al disco de ZimaOS (vía SMB u otro método):

```
/var/lib/casaos/apps/   →   directorio con un docker-compose.yml por app
```

Para cada app, importar el compose en ZimaOS (**App Store → Import compose**) con estos ajustes:

1. Eliminar la sección `x-casaos:` al final del compose (ZimaOS no la entiende)
2. Ajustar las rutas de volúmenes:
   - `/DATA/...` → `/DATA/...` (si ZimaOS tiene `/DATA` montado en el disco)
   - `/srv/lsio/...` → `/media/HDD-Storage/srv/lsio/...` (rutas personalizadas)

### Ejemplo de compose limpio para ZimaOS

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

## Estructura de carpetas resultante en ZimaOS

```
/DATA/                          ← Disco grande (HDD-Storage montado)
├── AppData/                    ← Configuraciones de apps
├── Downloads/
├── Documents/
├── Gallery/
├── Media/
│   ├── Movies/
│   └── TV Shows/
└── srv/
    └── lsio/                   ← Configuraciones de contenedores linuxserver
        ├── jellyfin/
        ├── radarr/
        ├── prowlarr/
        └── duplicati/
```

---

## Notas importantes

- ZimaOS (Buildroot) **no tiene `lvm2`**, por eso todo el trabajo de activar el LVM se hace desde Proxmox.
- El disco de datos queda montado en ZimaOS en `/media/HDD-Storage` y también accesible como `/DATA`.
- Los compose de CasaOS incluyen metadatos en `x-casaos:` que hay que eliminar antes de importar en ZimaOS.
- Los contenedores de linuxserver (`/srv/lsio/`) guardan su config fuera de `/DATA`, hay que usar la ruta completa al importar.
- Para contenedores linuxserver, el volumen `/config` debe apuntar a la carpeta **padre** que contiene `config/`, `data/`, `cache/`, etc., no a la subcarpeta `config/` directamente.

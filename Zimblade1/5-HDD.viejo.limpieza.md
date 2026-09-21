
# Reutilización de disco viejo (Windows 8) como ZFS pool en Proxmox

**Nodo:** `pve` (nodo1)
**Disco:** `/dev/sda` — ST1000LM024, 931.5GB (HDD 2.5" de portátil, antes con Windows 8)
**Storage creado:** `ST1000LM024` (zfspool)

## Contexto

Disco de un portátil antiguo con Windows 8 (7 particiones: EFI, recovery, MSR, NTFS principal, etc). Se extrajeron los datos importantes previamente. Objetivo: limpiar por completo y reutilizar como storage ZFS en Proxmox.

## 1. Identificación del disco

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TYPE
fdisk -l /dev/sda
mount | grep sda   # confirmar que no está montado nada
```

## 2. Borrado completo del disco

```bash
wipefs -a /dev/sda
dd if=/dev/zero of=/dev/sda bs=1M count=100
dd if=/dev/zero of=/dev/sda bs=1M count=100 seek=$(( $(blockdev --getsz /dev/sda) / 2048 - 100 ))
```

Verificación (debe salir vacío, sin firmas ni particiones):

```bash
lsblk /dev/sda
wipefs /dev/sda
```

## 3. Creación del zpool

Identificar el disco por ID persistente (evita problemas si cambia el nombre `/dev/sdX` entre reinicios):

```bash
ls -l /dev/disk/by-id/ | grep sda
```

Crear el pool:

```bash
zpool create -o ashift=12 ST1000LM024 /dev/disk/by-id/ata-ST1000LM024_HN-M101MBB_S31LJ9HDB00301
```

Activar compresión (lz4, bajo coste de CPU, buena relación compresión/rendimiento en HDD):

```bash
zfs set compression=lz4 ST1000LM024
```

Verificación:

```bash
zpool status ST1000LM024
```

## 4. Añadir como storage en Proxmox

Vía interfaz web: **Datacenter → Storage → Add → ZFS**

* ID: `ST1000LM024`
* ZFS Pool: `ST1000LM024`
* Content: Disk image, Container (según necesidad)
* **Nodes: solo `pve`** (¡importante! Si se deja "All (No restrictions)", aparece en otros nodos del clúster como `pve2` sin poder usarse ahí, ya que el disco físico solo existe en `pve`)

Equivalente por CLI:

```bash
pvesm add zfspool ST1000LM024 -pool ST1000LM024 -content images,rootdir,backup,iso,vztmpl -nodes pve
```

Corregir restricción de nodos si ya se creó sin especificar:

```bash
pvesm set ST1000LM024 -nodes pve
```

## 5. Verificación final

```bash
pvesm status
cat /etc/pve/storage.cfg | grep -A5 ST1000LM024
zfs list ST1000LM024
```

## Notas y aprendizajes

* **ZFS es storage local**: no se comparte automáticamente entre nodos del clúster. Cada nodo necesita su propio disco físico + su propio pool. Para compartir por red hace falta una capa adicional (NFS, Ceph, ZFS over iSCSI).
* **Thin provision**: permite crear discos de VM "virtualmente" más grandes de lo que hay disponible físicamente, similar a como ya funciona LVM-thin. Útil para flexibilidad, pero requiere vigilar el uso real (`zfs list`) para evitar quedarse sin espacio si varias VMs crecen a la vez.
* **Replicación ZFS entre nodos** (`pvesr` / Datacenter → Replication) solo funciona entre storages ZFS. Actualmente:

  * `pve` (nodo1): tiene ZFS (`ST1000LM024`) + LVM-thin (disco de sistema)
  * `pve2` (nodo2): solo tiene 1 disco, todo en LVM-thin, sin disco libre para ZFS
  * Pendiente: añadir un disco físico adicional a `pve2` para poder crear un pool ZFS ahí y configurar replicación de las VMs importantes.
* **LVM-thin vs ZFS**: LVM-thin no es "peor", es una filosofía distinta — más ligero en RAM, sin checksumming ni replicación nativa por red. Para el sistema de Proxmox y la mayoría de VMs está bien tal cual; ZFS aporta valor extra (integridad de datos, snapshots eficientes, replicación) donde realmente importa proteger algo.
* Comando útil para comprobar filesystem del sistema en cualquier nodo:

  ```bash
  df -Th /
  lsblk -f
  zpool list
  ```

## Próximos pasos (pendiente)

1. Conseguir disco adicional para `pve2`
2. Repetir limpieza + creación de zpool en `pve2`
3. Configurar replicación (`pvesr create-local-job`) de las VMs/LXCs consideradas críticas

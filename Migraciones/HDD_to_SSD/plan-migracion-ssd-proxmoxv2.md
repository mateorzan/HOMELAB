# Plan de Migración: HDD → SSD en Clúster Proxmox (pve + pve2)

## Contexto y motivación

El nodo `pve` presentó un fallo recurrente de backup (`vzdump` error `-61 - No data available`) causado por sectores dañados en el disco físico `/dev/sdb` (Seagate BarraCuda ST2000DM008/DMZ08, 2TB).

**Diagnóstico confirmado vía SMART:**
- `Current_Pending_Sector: 8` — sectores irrecuperables
- `Offline_Uncorrectable: 8`
- `Reported_Uncorrect`: subiendo activamente (33 → 35 tras un solo intento de `dd`)
- `Reallocated_Sector_Ct: 0` — el disco no está reasignando los sectores dañados por sí solo

**Causa raíz probable:** el disco es **SMR** (Shingled Magnetic Recording), confirmado para la serie BarraCuda 2TB/4TB/8TB (ST2000DM008 = ST2000DMZ08, mismo producto con distinto sufijo de canal de venta). Los discos SMR son inadecuados para cargas de escritura aleatoria sostenida como un thin pool LVM sirviendo discos de VMs activas — su diseño está pensado para escritura secuencial (backups, archivo de fotos/vídeos), no para uso como storage de sistema.

**Actualización — gestión con Amazon:** en vez de RMA con reemplazo físico de Seagate, Amazon resolvió con un **reembolso (parcial/total) sin necesidad de devolver el disco**. Esto significa que **el disco original se conserva** — no hay disco de reemplazo en camino. Cualquier HDD sano nuevo para el rol de almacenamiento de datos será una compra aparte, sin prisa, cuando se decida.

**Intentos de reparación del sector (resultado: irreparable, confirmado):**
- `dd` de lectura sobre el sector exacto (`1871035480`, sacado del log del kernel) → falló, `Input/Output error`, ni siquiera pudo leer para reescribir.
- `dd if=/dev/zero` (escritura directa forzada) → también falló con I/O error.
- `hdparm --repair-sector 1871035480 --yes-i-know-what-i-am-doing` → reportó "succeeded" pero el SMART confirmó que no cambió nada real.
- En los 3 intentos, `Reallocated_Sector_Ct` se mantuvo en 0 (nunca reasignó) y `Reported_Uncorrect` solo subió (33→35→39) sin ningún beneficio.
- **Conclusión: esos 8 sectores (1 bloque físico de 4K) están muertos de forma permanente.** No hay comando de Linux, Proxmox ni herramienta de fabricante (a nivel ATA estándar) que pueda revivirlos — el resto del disco (~2TB menos ese fragmento) sigue funcionando con normalidad.
- **Decisión: dejar de intentar repararlo.** Cada intento adicional solo suma desgaste sin beneficio.

**Mitigación aplicada — `fstrim` dentro del guest (ZimaOS, filesystem Btrfs):**
```bash
sudo fstrim -av
```
Liberó ~528GB en el disco de datos de la VM 105. Al desmapear ese espacio del thin pool, el backup de PBS dejó de intentar leer la zona dañada y **completó con éxito por primera vez**. Este es el paso que desbloqueó el backup, sin necesidad de excluir el disco.

**Monitorización activa confirmada:** `smartd` + Gotify ya alertan automáticamente de cambios en el disco (aviso recibido el 22 de julio, antes incluso del diagnóstico manual). No hace falta comprobación manual periódica — solo prestar atención si aparece un atributo nuevo (ej. `Current_Pending_Sector` subiendo por encima de 8, o `Reallocated_Sector_Ct` > 0), señal de que el daño se ha extendido más allá de la zona ya conocida.

**Decisión de arquitectura:** separar físicamente boot/sistema (SSD) de datos masivos (HDD), aplicando el tipo de disco correcto a cada carga de trabajo. Esto además reduce a casi cero la carga de escritura aleatoria sobre el HDD, ralentizando su desgaste futuro (aunque sin garantía de vida útil concreta).

---

## Arquitectura objetivo (por nodo)

- **1x SSD** (WD Blue SA510 250GB, TLC) → Proxmox host + discos de sistema/boot de VMs y LXC
- **1x HDD** (el actual, y tras el RMA el de reemplazo) → backups, fotos, vídeos, discos de datos masivos de VMs

ZFS elegido para el SO por su checksumming activo (detección temprana de corrupción silenciosa) y posibilidad de ampliar a mirror en el futuro sin reinstalar, vía:
```bash
zpool attach rpool <disco_actual> <disco_nuevo>
```

ARC de ZFS limitado a 1GB inicialmente (RAM actual: 8GB por nodo), ampliable cuando se suba a 16GB:
```bash
echo "options zfs zfs_arc_max=1073741824" >> /etc/modprobe.d/zfs.conf
update-initramfs -u
reboot
```

---

## Fase 0 — Preparación

- **Backup COMPLETO de todo lo que hay en `pve` vía PBS, sin exclusiones de disco** — esto es obligatorio con la nueva estrategia (ver Fase 6/7), ya que el HDD se va a borrar por completo, no se va a preservar el LVM-thin existente.
  - VM 105 (ZimaOs): confirmar que el backup incluye el disco de datos completo (`scsi2`), no solo boot/efidisk.
  - VM 107 (OPNsense): también reside en el mismo disco `sdb` — no olvidar su backup.
- No apagar/tocar nada más una vez confirmado el backup — dejar el disco en reposo hasta el día de la migración (un disco sin actividad no puede empeorar por sí solo).

---

## Fase 1 — Gestión de quórum del clúster (antes de tocar el nodo a migrar)

Con solo 2 nodos, si uno se apaga/reinstala, el otro pierde quórum (1 de 2 votos) y su `pmxcfs` puede quedar en solo lectura.

Ejecutar **desde el nodo que se queda activo**:
```bash
pvecm expected 1
pvecm delnode <nombre_nodo_a_migrar>
```

---

## Fase 2 — Preparación física

- Apagar el nodo.
- Desconectar el HDD (evita conflictos de nombre de storage y errores de selección de disco durante el instalador).
- Conectar el SSD nuevo en su lugar.

*(Límite de hardware: máximo 2 discos conectables por nodo — el layout final usa exactamente esos 2 slots: 1 SSD + 1 HDD).*

---

## Fase 3 — Conceptos ZFS aplicados

- **RAID0 en el instalador con 1 solo disco** = sin redundancia, simplemente la única opción posible con un disco.
- **Mirror (RAID1)** = posible en el futuro con `zpool attach`, pero requeriría un 3er disco físico (no hay slot libre actualmente salvo que el ZimaBlade tenga M.2 adicional, a confirmar por modelo).
- **ZFS sin mirror igual aporta valor:** detecta corrupción silenciosa vía checksums aunque no pueda auto-repararla sin redundancia — mejora real sobre LVM+ext4 tradicional.

---

## Fase 4 — Instalación de Proxmox

1. Arrancar instalador con solo el SSD conectado.
2. Target Harddisk → **ZFS (RAID0)**.
3. Hostname e IP: **mantener los mismos** que tenía el nodo antes (simplifica referencias en Tailscale, NPM, Cloudflare Tunnel, documentación).
4. Configuración básica (timezone, password, email).
5. Post-instalación: limitar ARC a 1GB (ver comandos en sección de arquitectura) + reboot.
6. Verificar: `zpool status` → `rpool` ONLINE sin errores.
7. **Cambiar repositorio a no-subscription** (evita error de `apt update` sin suscripción de pago):
```bash
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve <codename> pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update && apt full-upgrade -y
```
*(verificar `<codename>` de Debian correspondiente a la versión de Proxmox instalada)*

---

## Fase 5 — Reincorporar el nodo al clúster

```bash
# En el nodo recién instalado
pvecm add <IP_del_otro_nodo>

# En el otro nodo, revertir el ajuste de quórum
pvecm expected 2

# Verificar
pvecm status
```

No existe rol de "nodo maestro" en Proxmox — todos los nodos son pares una vez unidos.

**Detalles externos a revisar tras reinstalar (no afectan al clúster pero sí a servicios):**
- Tailscale generará nueva identidad de nodo aunque se mantenga la IP — reautenticar.
- La clave SSH del host cambia — limpiar `known_hosts` en clientes que se conecten.

---

## Fase 6 — Reconectar el HDD y reformatear por completo (estrategia final adoptada)

**Decisión final para `pve`:** a diferencia del planteamiento inicial (conservar el LVM-thin existente), se opta por **borrar el HDD por completo y reformatear en ext4**, aprovechando el proceso para marcar los sectores dañados conocidos como bad blocks permanentes. Esto es posible porque ya se cuenta con backup completo vía PBS (Fase 0) y con esta operación se resuelve de raíz la duda sobre "bloques prohibidos" que no tiene solución limpia en LVM-thin.

```bash
# Confirmar el disco correcto antes de nada
lsblk

# Borrar toda la estructura anterior (tabla de particiones, LVM, todo)
wipefs -a /dev/sdb

# Crear partición limpia (fdisk o parted, una sola partición ocupando todo el disco)
fdisk /dev/sdb
# → n (nueva), p (primaria), enter, enter, enter, w (escribir)

# Formatear en ext4 con test exhaustivo de lectura+escritura,
# esto marca automáticamente los sectores dañados conocidos como bad blocks permanentes
mkfs.ext4 -cc /dev/sdb1
```

**Nota:** `-cc` es un test exhaustivo (lectura + escritura) y puede tardar varias horas en un disco de 2TB. Es el paso que sustituye por completo la necesidad de `dmsetup` o técnicas de exclusión a nivel de bloque de dispositivo — ext4 gestiona su propia lista de bad blocks de forma nativa y sencilla.

Montar y añadir como storage:
```bash
mkdir /mnt/hdd-data
mount /dev/sdb1 /mnt/hdd-data
blkid /dev/sdb1   # obtener UUID para /etc/fstab
```
Añadir la entrada correspondiente en `/etc/fstab` con ese UUID, y en Proxmox: **Datacenter → Storage → Add → Directory**, path `/mnt/hdd-data`.

---

## Fase 7 — Restaurar VMs desde PBS

Con el HDD limpio y montado, restaurar completo desde el backup de PBS (Fase 0):

```bash
qmrestore <archivo_backup_105> 105 --storage hdd-data
qmrestore <archivo_backup_107> 107 --storage hdd-data
```

Opcional, una vez confirmado que todo arranca bien: mover el disco de boot de la VM 105 al SSD para aprovechar velocidad:
```bash
qm move-disk 105 scsi0 local-zfs
```

Arrancar ambas VMs y verificar funcionamiento completo antes de continuar.

**Nota sobre `pve2`:** esta estrategia de reformateo completo se decidió específicamente para `pve` por el historial de sectores dañados confirmados. Para `pve2` (sin este problema), sigue aplicando el planteamiento original de la Fase 8 — preservar el LVM-thin existente sin reformatear, ya que no hay necesidad de marcar bad blocks ahí.

---

## Fase 8 — Migración de pve2 (con PBS + Ghost + 2 LXCs)

**Consideración especial:** PBS corre como VM (VMID 102) dentro de `pve2`. Su datastore (`zfs_backup`) es un **zpool virtual creado por el propio guest OS de la VM PBS**, sobre un disco (`scsi1`) que a su vez vive en el LVM-thin del host — no hay zpool a nivel de host, todo es LVM-thin desde la perspectiva de `pve2`.

**Paso previo obligatorio — sacar configs antes de reinstalar** (PBS no estará disponible durante el proceso, así que no sirve como referencia si no se guarda antes):
```bash
mkdir -p ~/backup-configs
cp /etc/pve/qemu-server/*.conf ~/backup-configs/
cp /etc/pve/lxc/*.conf ~/backup-configs/
scp -r ~/backup-configs mateorzan@<IP_destino>:/ruta/destino
```

Resto del proceso idéntico a Fases 1-7, con un matiz de orden en la Fase 7:
1. Reconstruir primero la VM 102 (PBS) — apuntando `scsi0` y `scsi1` a los discos LVM-thin existentes.
2. Arrancar PBS — el zpool interno se automonta solo (gestionado por el guest, no por el host).
3. Verificar datastore `zfs_backup` con histórico de backups intacto.
4. **Solo entonces**, reconstruir Ghost y los 2 LXCs restantes, ya que sus backups dependen de que PBS esté operativo.

---

## Fase 9 — Verificación final y pendientes

**Checklist de cierre:**
- [ ] `pvecm status` en ambos nodos → quórum 2/2, ambos ONLINE
- [ ] Todas las VMs/LXC arrancan y responden correctamente
- [ ] `zpool status` (host) → `rpool` de cada SSD ONLINE sin errores
- [ ] Dentro de la VM PBS: `zpool status` (guest) → datastore sano, histórico visible
- [ ] Backup de prueba end-to-end desde cada VM crítica a PBS

**Pendientes que quedan abiertos tras la migración:**

1. **RMA Seagate (serial `ZFL8W4QH`)** — cuando llegue el reemplazo:
   - Migrar `vm-105-disk-2` del HDD viejo (dañado) al nuevo con `pvmove` o `qm move-disk`.
   - Retirar físicamente el HDD dañado.
   - Mientras tanto, monitorizar: `smartctl -a /dev/sdX | grep -E "Pending|Reallocated|Uncorrect"`.

2. **`lvm-thin-fix.service` (pve2)** — la race condition de boot original debería desaparecer al migrar el SO a ZFS (ya no depende de la activación de `data_tmeta`/`data_tdata` en el arranque). Confirmar con varios reinicios de prueba antes de cerrar definitivamente este pendiente en la documentación.

3. **Actualizar documentación** — repo GitHub (HOMELAB) y Notion: nueva arquitectura (SSD boot + HDD datos), decisión ZFS vs LVM-thin, lección aprendida sobre discos SMR y verificación de modelos antes de comprar.

---

## Lecciones aprendidas (para futuras compras de hardware)

- **HDD:** verificar SMR vs CMR contra la lista oficial antes de comprar: https://www.seagate.com/products/cmr-smr-list/. SMR es válido para backups/archivo, inadecuado para storage activo de VMs.
- **SSD:** priorizar NAND TLC sobre QLC para cargas de VM con escritura sostenida. Cuidado con modelos que han cambiado de TLC a QLC silenciosamente en revisiones posteriores del mismo nombre comercial (ej. Kingston A400).
- **Segunda mano:** aceptable con envío gestionado por Amazon + política de devolución amplia (30 días) + buenas reseñas del vendedor — verificar SMART/wear-leveling al recibir, dentro del plazo de devolución.

# Plan de Migración: HDD → SSD en Clúster Proxmox (pve + pve2)

## Contexto y motivación

El nodo `pve` presentó un fallo recurrente de backup (`vzdump` error `-61 - No data available`) causado por sectores dañados en el disco físico `/dev/sdb` (Seagate BarraCuda ST2000DM008/DMZ08, 2TB).

**Diagnóstico confirmado vía SMART:**
- `Current_Pending_Sector: 8` — sectores irrecuperables
- `Offline_Uncorrectable: 8`
- `Reported_Uncorrect`: subiendo activamente (33 → 35 tras un solo intento de `dd`)
- `Reallocated_Sector_Ct: 0` — el disco no está reasignando los sectores dañados por sí solo

**Causa raíz probable:** el disco es **SMR** (Shingled Magnetic Recording), confirmado para la serie BarraCuda 2TB/4TB/8TB (ST2000DM008 = ST2000DMZ08, mismo producto con distinto sufijo de canal de venta). Los discos SMR son inadecuados para cargas de escritura aleatoria sostenida como un thin pool LVM sirviendo discos de VMs activas — su diseño está pensado para escritura secuencial (backups, archivo de fotos/vídeos), no para uso como storage de sistema.

**RMA en curso con Seagate** — Serial: `ZFL8W4QH`, Modelo: `ST2000DM008-2UB102`.

**Decisión de arquitectura:** separar physically boot/sistema (SSD) de datos masivos (HDD), aplicando el tipo de disco correcto a cada carga de trabajo.

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

- Backup de todo lo posible vía PBS antes de tocar nada.
- **No tocar el HDD con datos** — se conserva intacto, no se rescata a un tercer sitio. El disco de reemplazo del RMA será el destino final para migrar los datos, evitando un paso intermedio innecesario.

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

## Fase 6 — Reconectar el HDD y recuperar sus datos (sin reformatear)

**Importante:** el HDD conserva sus datos intactos (VMs, discos de sistema y de datos) en su VG de LVM-thin original. No se reformatea ni se trata como storage vacío — se reimporta tal cual.

```bash
vgscan
vgchange -ay <nombre_VG>
```

Añadir en Proxmox: **Datacenter → Storage → Add → LVM-Thin**, seleccionando el VG/pool ya existentes. Proxmox debería listar los volúmenes ya presentes.

---

## Fase 7 — Reconstruir configuración de VMs/LXC

Como los datos siguen físicamente en el HDD, **no hace falta restaurar desde backup** — solo reconstruir el archivo de configuración (`.conf`) que le dice a Proxmox cómo ensamblar cada VM/LXC a partir de discos ya existentes.

```bash
qm create <VMID> --name <nombre> --memory <RAM> --cores <cores> \
  --scsi0 <storage>:vm-<VMID>-disk-1 \
  --scsi2 <storage>:vm-<VMID>-disk-2 \
  --efidisk0 <storage>:vm-<VMID>-disk-0
```

Valores de RAM/cores/red se consultan del backup de PBS (como referencia de configuración, no para restaurar discos) o de los `.conf` guardados manualmente antes de reinstalar (ver Fase 8 para el caso de `pve2`, donde esto es obligatorio por depender de PBS).

Arrancar y verificar funcionamiento antes de continuar.

*(Opcional, más adelante: mover el disco de boot al SSD con `qm move-disk <VMID> scsi0 local-zfs` para aprovechar velocidad, una vez todo esté estable).*

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

## Backups Redudency

Quiero tener mis Backups de mis maquinas en diferentes equipos por si mi disco se corrompe para ello vamos a usar la Rasp que tiene un disco SSD externo de 1TB con espacio de sobra para almacenar un historial de copias de seguridad de mis maquinas mas pequeñas. Esto es importante ya que mi PBS corre en PVE2 como una VM y mis LXCs y una VM corren en PVE2 por lo que si el disco se corrompe las copias de seguridad no servirian de nada ya que ya viven en ese disco.

### NFS

La version facil de todo esto es crear un almacenmiento en red por ejemplo NFS en mi rasp, conectarlo a Promox VE y crear ejecuciones de Backups a mayores que manden estas Backups a mi Raspberry, tambien vamos a aplicar esta metodologia ya que ahora mismo me sobra espacio en mi disco.

#### Ventajas

* Facilidad de configuracion y comodidad
* Rapidez
* Configurado todo desde la interfaz web de Proxmox
* Restauracion de Backups desde la propia interfaz de Proxmox sin necesidad de comandos

#### Desventajas

* No son incrementales, es decir siempre copia la backup desde 0
* Ocupa mucho mas espacio por lo que puedes mantener menos Backups lo que resulta en un menor historial
* Mas carga para el Proxmox ya que ahora tiene que hacer el doble de Backups.

Para hacer esto simplemente vamos a configurar un script bash que haga un restore de las copias de seguirdad que nosotros queramos, en mi caso hago copias de seguridad diarias de mis maquinas, por lo que voy a configurar que en mi rasp se guarden hasta 5 backups de cada maquina asi tengo un historial de 5 días de cada maquina.

#### Configuracion

En la raspberry pi instalamos y creamos la carpeta compartida por nfs

```Shell
sudo apt install nfs-kernel-server # Instalamos NFS
sudo nano /etc/exports # archivo donde exportamos la ruta por nfs

/ruta/backups 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash) # Ruta compartida en toda tu red Local

sudo exportfs -ra # exportamos la ruta
sudo systemctl restart nfs-kernel-server # Reiniciamos el servidor para que se aplique
```

#### Añadimos el Storage en Proxmox VE

En la interfaz web de Proxmox VE → Datacenter → Storage

  Click Add NFS:

   Rellena:

     ID: el nomrbe que tu quieras en mi caso rasp_bks

     **Server**: añades la IP de tu maquina.

   **Export**: ruta nfs compartida

   **Content**: Backup

**Guarda** — te va a mostrar el storage en tus PVEs

#### Crear tareas Backup

En la interfaz web de Proxmox VE → Datacenter → Backup

  Click Add NFS:

   Rellena:

     Storage: el nomrbe que tu quieras en mi caso rasp_bks

     **Schedule**: el que tu consideres

   **Selection Mode**: eliges la vm o lxc que quieras.

   **Compresion**: ZSTD

   **Mode**: Snapshot

**Create** — te va a mostar la tareada creada

Ahora haz esto con todas las VMs o LXCs que quieras.

#### Comprobación

Como ultimo paso puedes ir a la ruta que compartiste en tu maquina y ver como se guardan las backups alli.

![1788779180766](image/Raspberry_pi5/1788779180766.png)

### Script Bash

Esta es la segunda forma o version requiere de mas configuracion pero tiene ciertas ventajas

#### Ventajas

* Aqui las copias si son incrementales ya que usamos Borg.
* Más Control y capacidad de tener un historial más grande.
* Saca las copias del propio PBS por lo que no carga tanto el Proxmox VE.

#### Desventajas

* Mucha más configuración y mucho mas complejo.
* No se puede restaurar las backups desde la interfaz de Proxmox.
* Más incomodo cuando necesites recuperar una maquina.

Para hacer esto simplemente vamos a configurar un script bash que haga un restore de las copias de seguirdad que nosotros queramos, en mi caso hago copias de seguridad diarias de mis maquinas, por lo que voy a configurar que en mi rasp se guarden hasta 5 backups de cada maquina asi tengo un historial de 5 días de cada maquina.

El primer paso es configurar y crear el script bash este es el script que va a correr en la Raspberry y va a hacer el pull de los datos de las VMs o LXCs indicadas a la hora que yo indique. El unico requisito que tenemos que configurar es una clave SSH para que se conecten sin interrupciones.

antes de crear el archivo vamos a crear el .env que va a almacenar todas nuestras credenciales.

`nano .env`

```Shell
PBS_HOST="mateo@ip-vm-pbs"
PBS_REPO="mateo@pam!rasp-backup@localhost:zfs_bk"
export PBS_PASSWORD="el-secret-del-token"
export PBS_FINGERPRINT="AA:BB:CC:DD:...(el que copiaste)"
```

Lo portegemos para que solo nosotros podamos usarlo

`chmod 600 /ruta/backups/.env`

Importante crear el archivo como .sh

`nano script.sh`

```Shell
#!/bin/bash
set -euo pipefail

# cargar credenciales
source /ruta/backups/.env

DEST_BASE="/ruta/backups"
TMP_BASE="/tmp/pbs-restore"
KEEP=5

ITEMS=("ct/101" "ct/103" "ct/104" "vm/106")

for ITEM in "${ITEMS[@]}"; do
  NAME=$(echo "$ITEM" | tr '/' '-')
  echo "=== $NAME ==="

  SNAPSHOT=$(ssh "$PBS_HOST" \
    "PBS_PASSWORD='$PBS_PASSWORD' proxmox-backup-client snapshot list $ITEM --repository $PBS_REPO --output-format json" \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print(sorted(d, key=lambda x: x['backup-time'])[-1]['backup-time'])")

  if [ -z "$SNAPSHOT" ]; then
    echo "  ! Sin snapshot, saltando"
    continue
  fi
  echo "  snapshot: $SNAPSHOT"

  FILES=$(ssh "$PBS_HOST" \
    "PBS_PASSWORD='$PBS_PASSWORD' proxmox-backup-client snapshot files ${ITEM}/${SNAPSHOT} --repository $PBS_REPO --output-format json" \
    | python3 -c "import json,sys; [print(f['filename']) for f in json.load(sys.stdin) if not f['filename'].startswith('client.log') and not f['filename'].startswith('index.json')]")

  TMP_DIR="$TMP_BASE/$NAME"
  rm -rf "$TMP_DIR"
  mkdir -p "$TMP_DIR"

  for FILE in $FILES; do
    echo "  restaurando: $FILE"
    ssh "$PBS_HOST" \
      "PBS_PASSWORD='$PBS_PASSWORD' proxmox-backup-client restore ${ITEM}/${SNAPSHOT} ${FILE} - --repository $PBS_REPO" \
      > "$TMP_DIR/$FILE"
  done

  REPO_PATH="$DEST_BASE/$NAME-repo"
  [ -d "$REPO_PATH" ] || borg init --encryption=none "$REPO_PATH"

  borg create "$REPO_PATH::$(date +%Y-%m-%d_%H%M)" "$TMP_DIR"
  borg prune --keep-daily="$KEEP" "$REPO_PATH"

  rm -rf "$TMP_DIR"
  echo "  OK"
done
```

#### Requisitos

* SSH & SSH Key
* Borg
* API Token en PBS
* Python3
* Fingerprit del certificado
* PV

### Instalar paquetes

Es necesario tener estos paquetes instalados en tu Raspberry

```Shell
sudo apt update && sudo apt install pv borg python3
```

### SSH Key

Vamos a generar la key en la Rasp

> `ssh-keygen -t ed25519 -C "rasp-backup"`

Luego la copiamos en nuestro PBS

> `ssh-copy-id mateo@pbs-vm-ip`

Por ultimo comprobamos

> `ssh mateo@pbs-vm-ip "echo ok"`

### API Token

#### Cómo crearlo

En la interfaz web de PBS → Configuration → Access Control → API Token

  Click Add

   Rellena:

     **User**: root@pam (o crea un usuario dedicado solo para esto, más limpio)

     **Token Name**: algo como rasp-backup

**Guarda** — te va a mostrar el secret del token una sola vez, cópialo ya que no se puede volver a ver.

#### Dar permisos de lectura al token

 Ve a Access Control → Permissions, añade una entrada:

   **Path**: /datastore/zfs_backup

   **API Token:** selecciona el que creaste (root@pam!rasp-backup)

   **Role**: DatastoreReader (permite listar snapshots y leer/restaurar, pero no borrar ni modificar)

### Certificado

Desde pbs ejecutamos

`proxmox-backup-manager cert info`

Y buscamos algo tipo AA:BB:CC... en la linea Fingerprint (sha256):

### Creamos ejecutable

Ahora que ya tenemos el script bien configurado y el ssh creamos el ejecutable.

> `chmod +x /ruta/a/tu/script.sh`

Por ultimo antes de configurar el cron hacemos una prueba y vemos que funciona bien el ejecutable, como es una prueba solo vasmos a provar con ct/101, haz una copia del script original y edita el script para que solo haga la backup con ct/101

```Shell
cp script.sh script_bk.sh
nano script.sh
bash /ruta/a/tu/script.sh
```

Una vez se termina de ejecutar ya podemos poner todas las VMs o LXCs que queramos copiar.

### Cron

Una vez creado y viendo que nos podemos conectar y que funciona ya lo tenemos todo ahora solo activar el cron para que se ejecute los dias y a las horas que queramos, en mi caso sera todos los dias a las 8 de la mañana.

# Tutorial de restauración — Backups PBS guardados en la Raspberry Pi

Este documento cubre cómo recuperar un CT o una VM a partir de los backups
guardados con `backup-pbs-rasp.sh` en la Raspberry Pi (repos de borg), para
el escenario en que el PBS principal no está disponible.

Aplica a: `ct-101`, `ct-103`, `ct-104`, `vm-106` (backups ligeros guardados
en `~/Backup/<nombre>-repo`).

---

## 0. Antes de empezar

* Necesitas acceso SSH/consola al nodo Proxmox VE donde vas a restaurar.
* Necesitas `borg` instalado en la Raspberry Pi (ya lo tienes).
* Necesitas la herramienta `pxar` en el nodo Proxmox para restaurar CTs
  (viene incluida en Proxmox VE por defecto).

---

## 1. Ver qué copias tienes disponibles

En la Raspberry Pi:

```bash
borg list ~/Backup/<nombre>-repo
```

Ejemplo:

```bash
borg list ~/Backup/ct-101-repo
```

Te da una lista de snapshots tipo `2026-09-06_1835`, `2026-09-05_1902`, etc.
Elige el que quieras restaurar (normalmente el más reciente).

---

## 2. Extraer el snapshot elegido

```bash
mkdir -p ~/Backup/restore-tmp/<nombre>
cd ~/Backup/restore-tmp/<nombre>
borg extract ~/Backup/<nombre>-repo::<snapshot>
```

Ejemplo:

```bash
mkdir -p ~/Backup/restore-tmp/ct-101
cd ~/Backup/restore-tmp/ct-101
borg extract ~/Backup/ct-101-repo::2026-09-06_1835
```

Dentro encontrarás archivos como:

* **CT:** `pct.conf.blob`, `root.pxar.didx`, `catalog.pcat1.didx`
* **VM:** `qemu-server.conf.blob`, `drive-scsi0.img.fidx` (o el nombre del disco que tengas)

> **Importante:** aunque el nombre incluya `.didx`/`.fidx`, el contenido ya
> es el archivo real y completo (fue reconstruido en el momento del backup
> con `proxmox-backup-client restore`), no un índice vacío.

Renombra quitando el sufijo, para que las herramientas de Proxmox lo reconozcan:

```bash
# CT
mv root.pxar.didx root.pxar

# VM (ajusta el nombre de disco si no es scsi0)
mv drive-scsi0.img.fidx drive-scsi0.img
```

---

## 3. Copiar los archivos al nodo Proxmox

```bash
scp -r ~/Backup/restore-tmp/<nombre> root@<ip-proxmox>:/tmp/restore-<nombre>/
```

---

## 4. Restaurar un CT

**4.1 — Crea un CT nuevo vacío** con un template base (mismo OS que el original si lo sabes; si no, cualquiera sirve como base, se sobrescribe el contenido igualmente):

```bash
pct create 999 local:vztmpl/debian-12-standard_*.tar.zst \
  --rootfs local-lvm:8 \
  --hostname restaurado \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp
```

(Ajusta `999` a un ID libre, y los recursos rootfs/red según lo que necesites.)

**4.2 — Arranca una vez y detenlo** (esto asegura que el rootfs está inicializado):

```bash
pct start 999
pct stop 999
```

**4.3 — Extrae el contenido real por encima del rootfs:**

```bash
pxar extract /tmp/restore-<nombre>/root.pxar /var/lib/lxc/999/rootfs/ --overwrite
```

**4.4 — Ajusta la configuración** comparando con la original.
El archivo `pct.conf.blob` tiene el texto de configuración con un pequeño
header binario delante. Para verlo:

```bash
tail -c +9 /tmp/restore-<nombre>/pct.conf.blob
```

(el `+9` salta el header; si el texto sale con basura al principio, prueba
`+13` o revisa a ojo dónde empieza el texto legible tipo `arch: amd64`)

Copia los valores relevantes (memoria, CPU, red, mount points) a la config
real del CT nuevo:

```bash
nano /etc/pve/lxc/999.conf
```

**4.5 — Arranca el CT:**

```bash
pct start 999
```

---

## 5. Restaurar una VM

**5.1 — Crea una VM nueva vacía:**

```bash
qm create 999 --name restaurada --memory 2048 --net0 virtio,bridge=vmbr0
```

(Ajusta memoria/red a lo que corresponda; revisa `qemu-server.conf.blob`
igual que en el paso 4.4 para saber los valores originales.)

**5.2 — Importa el disco:**

```bash
qm importdisk 999 /tmp/restore-<nombre>/drive-scsi0.img local-lvm
```

**5.3 — Asocia el disco importado a la VM:**

```bash
qm set 999 --scsi0 local-lvm:vm-999-disk-0
qm set 999 --boot order=scsi0
```

**5.4 — Arranca la VM:**

```bash
qm start 999
```

---

## 6. Ver la config original (referencia rápida)

Para leer cualquier archivo `.conf.blob` (tanto CT como VM), el patrón es
el mismo: saltar el header binario y quedarte con el texto:

```bash
tail -c +9 archivo.conf.blob
```

---

## 7. Limpieza

Cuando termines, borra los temporales tanto en la Rasp como en Proxmox:

```bash
# En la Rasp
rm -rf ~/Backup/restore-tmp/<nombre>

# En Proxmox
rm -rf /tmp/restore-<nombre>
```

---

## Resumen ultra-rápido (cheat sheet)

```bash
# 1. En la Rasp: extraer
borg list ~/Backup/<nombre>-repo
mkdir -p ~/Backup/restore-tmp/<nombre> && cd ~/Backup/restore-tmp/<nombre>
borg extract ~/Backup/<nombre>-repo::<snapshot>
mv root.pxar.didx root.pxar          # CT
mv drive-scsi0.img.fidx drive-scsi0.img   # VM

# 2. Copiar a Proxmox
scp -r ~/Backup/restore-tmp/<nombre> root@<ip-proxmox>:/tmp/restore-<nombre>/

# 3a. CT
pct create 999 local:vztmpl/<template> --rootfs local-lvm:8
pct start 999 && pct stop 999
pxar extract /tmp/restore-<nombre>/root.pxar /var/lib/lxc/999/rootfs/ --overwrite
# ajustar /etc/pve/lxc/999.conf con pct.conf.blob
pct start 999

# 3b. VM
qm create 999 --name restaurada --memory 2048 --net0 virtio,bridge=vmbr0
qm importdisk 999 /tmp/restore-<nombre>/drive-scsi0.img local-lvm
qm set 999 --scsi0 local-lvm:vm-999-disk-0
qm set 999 --boot order=scsi0
qm start 999
```

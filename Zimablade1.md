tu se

# Set Up Zimablade1 Servidor Principal 💻

## Objetivo

Usaremos uno de los Zimablades como servidor principal, en este servidor vamos a ejecutar la VM con casaos y todos los servicios principales que se usan, Nextcloud, Jellyfin, etc... Este servidor sera el principal y estará disponible 24/7. La idea es en un futuro descentralizar este servidor y dividir la carga entre los dos servidores.

## Requisitos

* Zimablade con RAM 8gb
* Pincho USB 8gb
* Cable Ethernet
* Monitor y adaptador miniDP a DP
* HUB usb

## Instalación

Vamos a empezar con la instalación, lo primero que necesitamos es desde otro dispositivo es descargar la iso de nuestro nuevo sistema operativo, Proxmox. Para ello vamos a la pagina oficial de [Proxmox](https://www.proxmox.com/en/downloads), en ella entramos en el apartado de downloads, aquí tenemos los diferentes sistemas que hay disponibles en este caso vamos a usar Proxmox Virtual Environment, lo seleccionamos y dentro de el veremos las diferentes versiones, este proyecto fue creado con la ultima version actual, Proxmox VE 8.4 ISO.

### Disco de arranque

Una vez instalado nuestro archivo de instalación ISO necesitamos crear el dispositivo de arranque con el que realizaremos la instalación, para esto necesitamos una unidad de almacenamiento, por ejemplo, un USB. En este proyecto se uso un usb cualquiera de 8 gb, puedes usar cualquiera unidad extraible que supere el tamaño de la ISO.

Para poder crear nuestro dispositivo de arranque necesitamos un programa, en este caso utilizamos Rufus, lo puedes descargar en su pagina oficial [Rufus](https://rufus.ie/es/#download).

### Inicio de instalación

Una vez tenemos los pasos previos completados pasamos a la instalación del software para instalar el sistema operativo es como cualquier otro sistema operativo, arrancamos el servidor conectado a una pantalla, un teclado y con el disco de instalación, por esto es necesario el hub USB.

Mientras nuestro servidor inicia presionamos la tecla F2 asi entraremos en la BIOS, aquí tenemos que editar el orden de arranque y indicar que arranque con el disco de instalación USB o el que usaste para la instalación. Una vez seleccionado pulsamos F10 para salir guardando los cambios, el servidor se reiniciara y arrancara con la instalación.

### Proceso de instalación

Proxmox VE.

Seguimos el tutorial creado por los propios creadores de Zimablade ([ZimaBlade_proxmoxInstall](https://www.zimaspace.com/docs/es/zimaboard/ZimaBlades-Cluster-PVE-Makes-Your-Service-Migratable))

*El Hub si se inicia la zima con el no funciona hay q iniciar con el y mientras inicia meterlo.*

Creamos el usb de instalación con RUFUS y con esta configuración, es importante que el esquema de la partición sea MBR.

![1766319984358](image/README/1766319984358.png)

Es importante que en la BIOS cambiar el Boot Options Priorities y seleccionar el usb como primero ya que sino iniciara con el almacenamiento interno y no instalara.

![1766322140289](image/README/1766322140289.png)

Una vez configurado dentro de la BIOS en el ultimo menu seleccionamos save and exit con esto se nos guardara y se reiniciara solo.

![1766322179110](image/README/1766322179110.png)

Una vez iniciado nos saldrá el menu de instalación en nuestro caso seleccionaremos modo terminal y seguiremos esta configuración. **IMPORTANTE** NO INSTALAR PROXMOX EN EL DISCO SSD INTERNO HAY QUE INsTALARLO EN EL DISCO HDD EXTERNO.

Una vez termina de instalar se nos reiniciara y antes de que se inicie hay que quitar el usb de instalación para que inicie con el disco con el que hicimos la instalación. Ahora nos pedirá meternos en la web para hacer la instalación inicial.

## VM CasaOS

Una vez instalado Proxmox nos da una URL con la cual tenemos todo el panel de administración de el servidor.

```text
https://192.168.1.47:8006/
```

Aquí iniciaremos sesión con el usuario y contraseña creados durante la instalación, normalmente root.

![1766322669409](image/README/1766322669409.png)

Luego cambiamos el repositorio de Proxmox al No-Subscription, eliminando los siguientes repositorios.

![1768587272196](image/Zimablade2/1768587272196.png)

Y añadimos el No-Subscription.

![1768587323722](image/Zimablade2/1768587323722.png)

Luego actualizamos los repositorios.

![1768587384395](image/Zimablade2/1768587384395.png)

Y los upgradeamos.

![1768587421477](image/Zimablade2/1768587421477.png)

Primero antes de hacer nada añadimos los equipos a Tailscale que es una VPN gratuita que nos permite acceder a los equipos y que los equipos se vean entre si aunque no estén en la misma red interna. En este caso no seria necesario ya que tengo los equipos en la misma red local, pero esto añade disponibilidad a nuestro servidor y facilidades de añadir próximos dispositivos a la red y poder acceder a ellos desde fuera de la red local, sin tener que configurar Proxys. En nuestro caso nos conectamos a la shell de nuestro servidor pve y ejecutamos los siguientes comandos.

### Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Instalamos Tailscale.

```bash
sudo tailscale up
```

Activamos Tailscale y seguimos las indicaciones par vincularlo.

```bash
tailscale status
```

Puedes ver que esta bien configurado.

### Virtual Machine

En nuestro caso dentro de este servidor vamos a crear una Maquina Virtual que va a alojar un CasaOS con diferentes servicios, la configuración de la VM es la siguiente.

Creamos la VM, pulsamos el botón Create VM

Aquí seleccionamos el ID único de la maquina y su nombre, en este caso CasaOSZima1

![1772007202097](image/Zimablade1/1772007202097.png)

Luego vamos a la segunda sección OS, aquí tenemos que seleccionar un sistema operativo que nosotros elijamos, en este caso Ubuntu-24-04-03. Previamente a esto si no queremos complicaciones vamos a descargar la imagen ISO desde Promox, para ellos nos vamos a nuestro servidor, en este caso pve y dentro de el local (pve).

![1772007240410](image/Zimablade1/1772007240410.png)

Aquí podemos o subir la imagen o proporcionar la url con la imagen ISO para su descarga.

Una vez subida la ISO ya nos aparece para seleccionar la imagen ISO.

![1772007279229](image/Zimablade1/1772007279229.png)

En system vamos a seleccionar machine q35 y Disk vamos a dejar todo por defecto salvo el tamaño en nuestro caso tenemos hasta 2TB pero vamos a utilizar hasta 1.35TB.

![1772007351920](image/Zimablade1/1772007351920.png)

En CPU vamos a añadir dos cores, que es lo máximo que tiene disponible este servidor, el resto por defecto.

![1772007369750](image/Zimablade1/1772007369750.png)

En memory vamos a poner 7800 que equivale a 7.62gb de 8gb que tenemos disponibles.

![1772007386549](image/Zimablade1/1772007386549.png)

En Network lo vamos a dejar por defecto.

Con todo esto ya tenemos nuestra VM con Ubuntu24 para crear nuestro CasaOS, la configuración tendría que quedar algo asi.

![1766329574197](image/README/1766329574197.png)

### IP estática VM

Usar IP local fija en todos los servidores, ejemplo de como configurar en los Zimablades, cada equipo tiene su forma. También vamos a cambiar el hostname para identificarlo mejor y la contraseña del usuario por defecto.

```bash
hostnamectl
sudo hostnamectl set-hostname NUEVO_NOMBRE
sudo nano /etc/hosts
sudo passwd casaos
sudo passwd root
sudo reboot
```

Cambio de IP a IP fija.

```bash
sudo nmtui
```

Con este comando ya nos sale una interfaz con la que podemos configurar la IPV4 incluso aquí podemos configurar el hostname configurado anteriormente.

### CasaOS

Una vez creada nuestra VM vamos a crear nuestro servidor CasaOS en esta guía no se va a explicar como se creo esta estructura ya que se va a migrar un servidor casaOS ya creado, el cual estaba alojado en una Raspberry Pi5, a este servidor Proxmox nuevo. La migración se basa en copiar la carpeta /Data de nuestro antiguo servidor a este nuevo servidor, esto es simple ya que mi antiguo servidor tenia un disco SSD externo extraible, por lo cual es conectar este disco por USB a el nuevo servidor y seguir estos pasos.

Instalamos Casaos con el comando de instalación.

```bash
curl -fsSL https://get.casaos.io | sudo bash
```

Procedemos a copiar lo configuración de nuestro CasaOS de la raspberry Pi a este Zima, para ello conectamos el disco duro externo SSD a nuestro Zima y lo montamos en nuestra VM.

![1766334596629](image/README/1766334596629.png)

Ahora vamos a montar el disco y realizar el copiado con los siguientes comandos.

```bash
lsblk
```

Mostramos los discos y vemos que el disco externo esta montado

```bash
sudo mkdir -p /mnt/rpi
sudo mount /dev/sdb2 /mnt/rpi
```

Creamos la ruta para montar el disco externo y lo montamos en /mnt/rpi

```bash
lsblk -f
```

Si da error puedes ver mas información de los discos aquí.

```bash
sudo systemctl stop casaos
sudo systemctl stop docker
```

Paramos tanto docker como casaos para poder hacer el copiado.

```bash
sudo lvdisplay

sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

sudo resize2fs /dev/ubuntu-vg/ubuntu-lv

df -h
```

Antes de copiar vamos a extender el almacenamiento para que nos entre todo

```bash
sudo rsync -avh --progress /mnt/rpi/DATA/ /DATA/
sudo rsync -avh /mnt/rpi/var/lib/casaos/ /var/lib/casaos/
sudo rsync -avh /mnt/rpi/etc/casaos/ /etc/casaos/
sudo rsync -avh /mnt/rpi/srv/lsio/ /srv/lsio/
```

Hacemos el copiado de la carpetas de configuración y DATA del disco externo a la carpeta data del Zimablade1, copiamos esta carpeta ya que es la que contiene todos los datos y las configuraciones de los servicios el propio CasaOS no nos interesa ya que ya lo tenemos creado.

```bash
sudo chown -R root:root /DATA
sudo chown -R root:root /var/lib/casaos
sudo chown -R root:root /etc/casaos
sudo chown -R root:root /srv/lsio/
```

Le damos permisos de root a la carpeta por si acaso.

```bash
sudo systemctl start docker
sudo systemctl start casaos
```

Una vez todo funciono bien iniciamos Docker y CasaOS.

Desmontamos disco SSD externo

```bash
sudo lsof +D /mnt/rpi 2>/dev/null

sudo umount /mnt/rpi

lsblk
```

Una vez desmontado retiramos el SSD externo y reiniciamos la VM ahora debería de arrancar bien con el HDD.

## LXC Network-Services / Migrada hacia PVE2

Creamos esta LXC para descentralizar los servicios encargados de la exposición de ciertos servicios y asi sean mas accesibles y replicables el proceso de migración de estos servicios esta explicado en Migration.md de este Repositorio. Con esto vamos a aprovechar esta LXC para ademas crear un dashboard para poder ver que servicios hay en todo mi servidor y tener una vista general.

### Ddns-Updater

Actualiza la IP Pública la cual esta asociada a mi dominio.

### NPM

Gestor de URL públicas proxy, redirecciona las urls hacia los servicios que yo quiero que sean accesibles publicamente.

### Glance

Como dashboard elegí [Glance](https://github.com/glanceapp/glance?tab=readme-ov-file#installation) ya que me gusta su estética y sus funcionalidades, aunque no es un dashboard fácil de configurar. Vamos a seguir la instalación recomendad a traves de docker compose.

```bash
mkdir glance && cd glance && curl -sL https://github.com/glanceapp/docker-compose-template/archive/refs/heads/main.tar.gz | tar -xzf - --strip-components 2
```

Una vez instalado lanzamos el contenedor con docker compose.

```bash
docker compose up -d
```

Una vez lo tenemos corriendo podemos configurar a nuestra manera dentro de la carpeta `./glance/config/` ahi tenemos dos archivos que vienen ya con configuraciones de ejemplo `/home` es la pagina inicial y `glance` en la configuración del dashboard en general.

Yo borre esta configuración y cree la mia propia aunque aun sigue en proceso de construcción.

![1775650674189](image/Zimablade1/1775650674189.png)

Para acceder al dashboard es desde la url que hayas configurado en tu docker-compose, por ejemplo `http://192.168.1.xx:8080/`

La configuración de server stats lo hice tanto usando el binaria como usando el docker-compose.yml para obtener las métricas de los contenedores.

```bash
services:
  glance-agent:
    container_name: glance-agent
    image: glanceapp/agent:latest
    restart: unless-stopped
    # environment:
      # TOKEN: your_auth_token_here
    volumes:
      - /:/host:ro
      - /proc:/proc:ro
      - /sys:/sys:ro
      - /dev:/dev:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /etc/os-release:/etc/os-release:ro
    ports:
      - "27973:27973"
```

### Homepage Dashboard

Vamos a crear un Dashboard para nuestro Homelab

#### Docker

Instalación con docker compose.

```
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - 3000:3000
    volumes:
      - ./config:/app/config # Make sure your local config directory exists
      - /var/run/docker.sock:/var/run/docker.sock:ro # (optional) For docker integrations
    environment:
      HOMEPAGE_ALLOWED_HOSTS: 192.168.1.55:3000 # required, may need port. See gethomepage.dev/installation/#homepage_allowed_hosts
    restart: unless-stopped
```

## OPNsense VM / DEPECRATED

Ya empiezo a tener bastantes servicios y contenedores por lo que le voy a añadir una casa de seguridad más grande a mi red, por ello vamos a instalar un VM con [OPNsense]('https://opnsense.org/#'), un firewall Open-source muy potente.

![1778061173417](image/Zimablade1/1778061173417.png)

Como en nuestro caso vamos a crear un VM en Proxmox vamos a utilizar la imagen ISO que nos proporciona OPNsense en su web, una vez descargada y descomprimida subimos la ISO a uno de nuestros storages, yo en mi caso lo voy a subir en el storage local que viene por defecto.

![1778061877363](image/Zimablade1/1778061877363.png)

### Requisitos

* 1 core
* 4gb de ram o más
* 8gb de almacenamiento en disco o más

![1778062123943](image/Zimablade1/1778062123943.png)

### Instalación

![1778062468167](image/Zimablade1/1778062468167.png)

Ahora una vez inicia nos da estas dos opciones , como nosotros queremos instalar vamos a loguear con 'installer', y la contraseña por defecto es 'opnsense'.

Primero elegimos el idioma del teclado, como tengo teclado español vamos con Spanish

![1778062846007](image/Zimablade1/1778062846007.png)

Ahora vamos a instalar el sistema de archivos, ya que es una VM con pocos recursos vamos a ir por la opción UFS.

![1778063031282](image/Zimablade1/1778063031282.png)

Importante ahora seleccionar el disco no el cd, que es realmente la ISO de instalacion.

![1778063082713](image/Zimablade1/1778063082713.png)

Ahora instalara el OPNsense en nuestro disco de 20gb que asignamos a nuestra VM.

![1778063533507](image/Zimablade1/1778063533507.png)

Una vez instalado nos va a pedir cambiar la contraseña, la cambiamos y podemos completar la instalación.

![1778064379672](image/Zimablade1/1778064379672.png)

Una vez la completamos reiniciamos el sistema y quitamos el disco CD/ISO, y ahora nos iniciara desde el disco donde realizamos la instalación.

Ahora ya podemos iniciar sesión con root y la contraseña que configuramos anteriormente.

![1778064861621](image/Zimablade1/1778064861621.png)

### Configuración

Ahora lo primero que vamos a hacer es configurar la IP, el la opción 2).

![1778064959655](image/Zimablade1/1778064959655.png)

Asi es como configure la red IPV4

![1778065269240](image/Zimablade1/1778065269240.png)

Una vez tenemos configurada la IP del host ya podemos acceder a la GUI web de OPNsense

![1778065487707](image/Zimablade1/1778065487707.png)

Una vez dentro vamos a ir al wizard y vamos a revisar la configuración básica.

![1778066810911](image/Zimablade1/1778066810911.png)

## VM ZimaOS

Migración de CasaOS hacia ZimaOS, evolucion de este sofware con multiples mejoras y soporte actual, ya que casaos ya no recibia actualizaciones ni mejoras. En el apartado de migraciones esta documentado todo el proceso de migración y como consegui mover un disco de 800gb de una VM a otra con sus particularidades, ya que ZimaOS es bastante mas restrictivo que su version antigua CasaOS.

![1780744271486](image/Zimablade1/1780744271486.png)

### Ansible

Vamos a añadir un nuevo servicio a mi ZimaOS, ansible es una herramienta que automatiza la configuración de servidores, el despliegue de programas y la gestión de redes sin necesidad de instalar agentes externos en los equipos controlado. Esta tambien te ayuda a monitorizar los servidores que tengamos, actualizaciones, versiones, etc...

Vamos a aprovechar la App Store que nos ofrece ZimaOS y vamos a instalar Ansile Semaphore que es una version de Ansible CLI que ofrece una capa visual para gestionar todo de manera mas comoda via web.

![1787134360931](image/Zimablade1/1787134360931.png)

Vamos a entrar en la configuración en mi caso tengo el puerto 3000 ocupado por lo que voy a usar el puerto 3030

![1787135042813](image/Zimablade1/1787135042813.png)

Luego tambien recomiendo configurar las contraseñas y usuario y no dejarlas por defecto por un tema de seguridad pero si tu servidor no esta expuesto al exterior puedes dejarlo por defecto

![1787135116973](image/Zimablade1/1787135116973.png)

Luego una vez se instale la App accedemos a su web y iniciamos sesion

![1787136230030](image/Zimablade1/1787136230030.png)

Luego lo primero es crear el proyecto con el nombre que tu quieras en mi caso Homelab

![1787136552252](image/Zimablade1/1787136552252.png)

Yo ansible lo quiero principalmente para poder conectarme a mis servidores y poder lanzar playbooks desde aqui, por ejemplo me interesa poder lanzar actualizaciones de mis servidores semanales todo automatizado. Esto se puede hacer ya que Ansible se conecta por SSH a los servidores que tu quieras y lanza los playbooks, por esto lo primero es configurar las Keys, vamos a empezar con una prueba conectando mi Raspberry Pi 5

![1787137994145](image/Zimablade1/1787137994145.png)

Tambien es interesante e importante crear una key para la autentificacion del sudo ya que la mayoria de operaciones lo necesitan, simplemente añade la contraseña

![1787141095949](image/Zimablade1/1787141095949.png)

Luego tenemos que crear el repositorio para almacenar los playbooks yo voy a almacenarlos localmente

![1787139065927](image/Zimablade1/1787139065927.png)

Luego por ultimo configuramos el inventory que es donde especidficamos la IP y a donde se tiene que conectar

![1787141116285](image/Zimablade1/1787141116285.png)

Ahora vamos a crear el primer playbook yml, este va a ser un playbook que simplemente ejecute un apt update, como lo configuramos con un repo local tenemos que meternos a la terminal de nuestro contenedor y dentro de la estructura creada crear el archivo `apt-update.yml`

```Shell
cat > apt-update.yml << 'EOF'
---
- name: Actualizar repositorios APT
  hosts: raspberry
  become: yes
  tasks:
    - name: apt update
      apt:
        update_cache: yes
EOF
```

![1787139488703](image/Zimablade1/1787139488703.png)

Ahora a vamos al ultimo paso vamos a crear la Task Template esto es simplemente añadir todo lo que acabamos de crear

![1787139831464](image/Zimablade1/1787139831464.png)

Luego ejecutamos y vemos si funciona como deberia

![1787140987307](image/Zimablade1/1787140987307.png)

Por ultimo si queremos que sea una ejecucion periodica podemos añadirle un schedule pero esto es opcional

![1787141192705](image/Zimablade1/1787141192705.png)

Con esto ya tendriamos un ejemplo rapido de una ejecucion que se podria hacer con este servicio las posibilidades son infinitas.

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

![1787138291946](image/Zimablade1/1787138291946.png)

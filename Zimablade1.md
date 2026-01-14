# Set Up Zimablade1 servidor principal

## Objetivo

Usaremos uno de los zimablades como servidor principal, en este servidor vamos a ejecutar la VM con casaos y todos los servicios principales que se usan, Nextcloud, Jellyfin, etc... Este servidor sera el principal y estara disponible 24/7.

## Requisitos

* Zimablade con RAM 8gb
* Pincho USB 8gb
* Cable ethernet
* Monitor y adaptador miniDP a DP
* HUB usb

## Instalación

### Software

Vamos a empezar con la instalación, lo primero que necesitamos es desde otro dispositivo es descargar la iso de nuestro nuevo sistema operativo, Proxmox. Para ello vamos a la pagina oficial de Proxmox(https://www.proxmox.com/en/downloads), en ella entramos en el apartado de downloads, aqui tenemos los diferentes sistemas que hay disponibles en este caso vamos a usar Proxmox Virtual enviroment, lo seleccionamos y dentro de el veremos las diferentes versiones, este proyecto fue creado con la ultima version actual, Proxmox VE 9.1 ISO.

### Disco de arranque

Una vez instalado nuestro archivo de instalacion ISO necesitamos crear el dispositivo de arranque con el que realizaremos la instalacion, para esto necesitamos una unidad de almacenamiento, por ejemplo, un USB. En este proyecto se uso un usb cualquiera de 8 gb, puedes usar cualquiera unidad extraile que supere el tamaño de la ISO.

Para poder crear nuestro dispositivo de arranque necesitamos un programa, en este caso utilizamos Rufus, lo puedes descargar en su pagina oficial(https://rufus.ie/es/#download). Una vez instalado conectamos el disco extraible a nuestro dispositivo y lo configuramos de la siguiente manera:

AÑADIR IMGEN.

Seleccionamos Empezar con esto ya se nos instalara y se nos creara el disco de arranque.

### Inicio de instalación.

Una vez tenemos los pasos previos completados pasamos a la instalacion del software para instalar el sistema opetativo es como cualquier otro sistema operativo, arrancamos el servidor conectado a una pantalla, un teclado y con el disco de instalación, por esto es necesario el hub USB.

Mientras nuestro servidor inicia presionamos la teclar F2(MIRAR SI ES ESA TECLA) asi entraremos en la BIOS, aqui tenemos que editar el orden de arranque y indicar que arranque con el disco de instalacion USB o el que usaste para la instalacion. Una vez seleccionado pulsamos F10 para salir guardando los cambios, el servidor se reiniciara y arrancara con la instalacion.

### Proceso de instalacion.

#### Proxmox

Seguimos el tutorial creado por los propies creadores de zimablade ([ZimaBlade_proxmoxInstall](https://www.zimaspace.com/docs/es/zimaboard/ZimaBlades-Cluster-PVE-Makes-Your-Service-Migratable))

El Hub si se inicia la zima con el no funciona hay q iniciar con el y mientras inicia meterlo.

![1766319984358](image/README/1766319984358.png)

Creamos el usb de instalacion con RUFUS y con esta configuracion, es importante que el esquema de la partición sea MBR.

![1766322140289](image/README/1766322140289.png)

Es importante que en la BIOS cambiar el Boot Options Priorities y seleccionar el usb como primero ya que sino iniciara con el almacenamiento interno y no instalara.

![1766322179110](image/README/1766322179110.png)

Una vez confugirado dentro de la BIOS en el ultimo menu seleccionamos save and exit con esto se nos guardara y se reiniciara solo.

Una vez iniciado nos saldra el menu de instalacion en nuestro caso seleccionaremos modo terminal y seguiremos esta configuracion. IMPORTANTE NO INSTARLAR PROXMOX EN EL DISCO SSD INTERNO HAY QUE INTALARLO EN EL DISCO HDD EXTERNO.

PONER FOTO

Una vez termina de instalar se nos reiniciara y antes de que se inicie hay que quitar el usb de instalacion para que incie con el disco con el que hicimos la instalacion. Ahora nos pedira meternos en la web para hacer la instalacion inicial.

![1766322669409](image/README/1766322669409.png)

## Configuración

Una vez instalado Proxmox nos da una URL con la cual tenemos todo el panel de administracion de el servidor.

```
https://192.168.1.47:8006/
```

Aqui iniciaremos sesion con el usuario y contraseña creados durante la instalacion, normalmente root.

Primero antes de hacer nada añadimos los equipos a tailscale que es una VPN gratuita que nos permite acceder a los equipos y que los equipos se vean entre si aunque no esten en la misma red interna. En este caso no seria necesario ya que tengo los equipos en la misma red local, pero esto añade disponibilidad a nuestro servidor y facilidades de añadir proximos dispositivos a la red y poder acceder a ellos desde fuera de la red local, sin tener que configurar proxys. En nuestro caso nos conectamos a la shell de nuestro servidor pve y ejecutamos los siguientes comandos.

### Tailscale

```
curl -fsSL https://tailscale.com/install.sh | sh
```

Instalamos Tailscale.

```
sudo tailscale up
```

Activamos Tailscale y seguimos las indicaciones par vincilarlo.

```
tailscale status
```

Puedes ver que esta bien configurado.

### IP FIJA

Usar IP local fija en todos los servidores, ejemplo de como configurar en los zimablades, cada equipo tiene su forma. Tambien vamos a cambiar el hostname para identificarlo mejor y la contraseña del usuario por defecto.

```
hostnamectl
sudo hostnamectl set-hostname NUEVO_NOMBRE
sudo nano /etc/hosts
sudo passwd casaos
sudo passwd root
sudo reboot
```

Cambio de IP a IP fija.

```
sudo nmtui
```

Con este comando ya nos sale una interfaz con la que podemos configurar la IPV4 incluso aqui podemos configurar el hostname configurado anteriormente.

### Virtual Machine

En nuestro caso dentro de este servidor vamos a crear una Maquina Virtual que va a alojar un CasaOS con diferentes servicios, la configuración de la VM es la siguiente.

Creamos la VM, pulsamos el boton Create VM

IMAGEN

Aqui seleccionamos el ID unico de la maquina y su nombre, en este caso CasaOSZima1

Luego vamos a la segunda seccion OS, aqui tenemos que seleccionar un sistema operativo que nosotros elijamos, en este caso Ubuntu-24-04-03. Previamente a esto si no queremos complicaciones vamos a descargar la imagen ISO desde promox, para ellos nos vamos a nuestro servidor, en este caso pve y dentro de el local (pve).

IMAGEN

Aqui podemos o subir la imagen o proporcionar la url con la imagen ISO para su descarga.

Una vez subida la ISO ya nos aparece para seleccionar la imagen ISO.

IMAGEN

En system vamos a selccionar machine q35

IMAGEN

En Disk vamos a dejar todo por defecto salvo el tamaño en nuestro caso tenemos hasta 2TB pero vamos a utilizar hasta 1.35TB.

IMAGEN

En CPU vamos a añadir dos cores, que es lo maximo que tiene disponible este servidor, el resto por defecto.

IMAGEN

En memory vamos a poner 7800 que equivale a 7.62gb de 8gb que tenemos disponibles.

IMAGEN

En Network lo vamos a dejar por defecto.

Creamos nuestro primera VM, vamos a crear un casaos para ello primero necesitamos un contenedor con sistema base debian con la siguiente configuración:

![1766329574197](image/README/1766329574197.png)

#### CasaOS

Una vez creada nuestra VM vamos a crear nuestro servidor CasaOS en esta guia no se va a explicar como se creo esta estructura ya que se va a migrar un servidor casaOS ya creado, el cual estaba alojado en una Raspberry Pi5, a este servidor Proxmox nuevo. La migración se basa en copiar la carptea /Data de nuestro antiguo servidor a este nuevo servidor, esto es simple ya que mi antiguo servidor tenia un disco SSD externo extraible, por lo cual es conectar este disco por USB a el nuevo servidor y seguir estos pasos.

Instalamos Casaos con el comando de instalacion.

```
curl -fsSL https://get.casaos.io | sudo bash
```

Procedemos a copiar lo configuracion de nuestro CasaOS de la raspberry Pi a este zima, para ello conectamos el disco duro externo SSD a nuestro zima y lo montamos en nuestra VM.

![1766334596629](image/README/1766334596629.png)

Ahora vamos a montar el disco y realizar el copiado con los siguientes comandos.

```
lsblk
```

Mostramos los discos y vemos que el disco externo esta montado

```
sudo mkdir -p /mnt/rpi
sudo mount /dev/sdb2 /mnt/rpi
```

Creamos la ruta para montar el disco externo y lo montamos en /mnt/rpi

```
lsblk -f
```

si da error puedes ver mas informacion de los discos aquí.

```
sudo systemctl stop casaos
sudo systemctl stop docker
```

Paramos tanto docker como casaos para poder hacer el copiado.

```
sudo lvdisplay

sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

sudo resize2fs /dev/ubuntu-vg/ubuntu-lv

df -h
```

Antes de copiar vamos a extender el almacenamiento para que nos entre todo

```
sudo rsync -avh --progress /mnt/rpi/DATA/ /DATA/
sudo rsync -avh /mnt/rpi/var/lib/casaos/ /var/lib/casaos/
sudo rsync -avh /mnt/rpi/etc/casaos/ /etc/casaos/
sudo rsync -avh /mnt/rpi/srv/lsio/ /srv/lsio/
```

Hacemos el copiado de la carpetas de configuracion y DATA del disco externo a la carpeta data del zimablade1, copiamos esta carpeta ya que es la que contiene todos los datos y las configuraciones de los servicios el propio CasaOS no nos interesa ya que ya lo tenemos creado.

```
sudo chown -R root:root /DATA
sudo chown -R root:root /var/lib/casaos
sudo chown -R root:root /etc/casaos
sudo chown -R root:root /srv/lsio/
```

Le damos permisos de root a la carpeta por si acaso.

```
sudo systemctl start docker
sudo systemctl start casaos
```

Una vez todo funciono bien iniciamos Docker y CasaOS.

Desmontamos disco SSD externo

sudo lsof +D /mnt/rpi 2>/dev/null

sudo umount /mnt/rpi

lsblk

Una vez desmontado retiramos el SSD externo y reiniciamos la VM ahora deberia de arrancar bien con el HDD.

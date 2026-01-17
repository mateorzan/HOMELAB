# Set Up Zimablade2 servidor secundario/BACKUP

## Objetivo

Usaremos uno de los zimablades como servidor secundario, en este servidor vamos a ejecutar las copias de seguridad diarias para asegurar que no se pierda informacion, ademas tambien replicaremos la VM por si el servidor principal cae este lo respalde.

## Requisitos

* Zimablade con RAM 8gb
* Pincho USB 8gb
* Cable ethernet
* Monitor y adaptador miniDP a DP
* HUB usb

## Preparación Instalación

### Software

Vamos a empezar con la instalación, lo primero que necesitamos es desde otro dispositivo es descargar la iso de nuestro nuevo sistema operativo, Proxmox. Para ello vamos a la pagina oficial de Proxmox(https://www.proxmox.com/en/downloads), en ella entramos en el apartado de downloads, aqui tenemos los diferentes sistemas que hay disponibles en este caso vamos a usar Proxmox Virtual enviroment, lo seleccionamos y dentro de el veremos las diferentes versiones, este proyecto fue creado con la ultima version actual, Proxmox VE 8.4 ISO.

### Disco de arranque

Una vez instalado nuestro archivo de instalacion ISO necesitamos crear el dispositivo de arranque con el que realizaremos la instalacion, para esto necesitamos una unidad de almacenamiento, por ejemplo, un USB. En este proyecto se uso un usb cualquiera de 8 gb, puedes usar cualquiera unidad extraile que supere el tamaño de la ISO.

Para poder crear nuestro dispositivo de arranque necesitamos un programa, en este caso utilizamos Rufus, lo puedes descargar en su pagina oficial(https://rufus.ie/es/#download).

## Proceso de instalacion.

#### Proxmox

Seguimos el tutorial creado por los propies creadores de zimablade ([ZimaBlade_proxmoxInstall](https://www.zimaspace.com/docs/es/zimaboard/ZimaBlades-Cluster-PVE-Makes-Your-Service-Migratable))

*El Hub si se inicia la zima con el no funciona hay q iniciar con el y mientras inicia meterlo.*

Creamos el usb de instalacion con RUFUS y con esta configuracion, es importante que el esquema de la partición sea MBR.

![1766319984358](image/README/1766319984358.png)

### Inicio de instalación.

Una vez tenemos los pasos previos completados pasamos a la instalacion del software para instalar el sistema opetativo es como cualquier otro sistema operativo, arrancamos el servidor conectado a una pantalla, un teclado y con el disco de instalación, por esto es necesario el hub USB.

Mientras nuestro servidor inicia presionamos la teclar F2(MIRAR SI ES ESA TECLA) asi entraremos en la BIOS, aqui tenemos que editar el orden de arranque y indicar que arranque con el disco de instalacion USB o el que usaste para la instalacion. Una vez seleccionado pulsamos F10 para salir guardando los cambios, el servidor se reiniciara y arrancara con la instalacion.

Es importante que en la BIOS cambiar el Boot Options Priorities y seleccionar el usb como primero ya que sino iniciara con el almacenamiento interno y no instalara. Para entrar en la BIOS pulsamos DEL(Supr).

![1766322140289](image/README/1766322140289.png)

Una vez confugirado dentro de la BIOS en el ultimo menu seleccionamos save and exit con esto se nos guardara y se reiniciara solo.

![1766322179110](image/README/1766322179110.png)

Una vez iniciado nos saldra el menu de instalacion en nuestro caso seleccionaremos modo graphical y seguiremos esta configuracion. IMPORTANTE NO INSTARLAR PROXMOX EN EL DISCO SSD INTERNO HAY QUE INTALARLO EN EL DISCO HDD EXTERNO.

# PONER FOTO

SI VES QUE CON EL GRAFICO NO TE FUNCIONA PRUEBA CON EL TERMINAL CON LA MISMA CONFIGURACIÓN.

> *Una vez termina de instalar se nos reiniciara y antes de que se inicie hay que quitar el usb de instalacion para que incie con el disco con el que hicimos la instalacion.*

Ahora nos pedira meternos en la web para hacer la instalacion inicial.

## Configuración

Una vez instalado Proxmox nos da una URL con la cual tenemos todo el panel de administracion de el servidor.

```
https://192.168.1.49:8006/
```

Aqui iniciaremos sesion con el usuario y contraseña creados durante la instalacion, normalmente root.

![1766322669409](image/README/1766322669409.png)

Luego cambiamos el repositorio de Proxmox al No-Subcripcion, eliminando los siguientes repositorios.

![1768587272196](image/Zimablade2/1768587272196.png)

Y añadimos el No-Subcription.

![1768587323722](image/Zimablade2/1768587323722.png)

Luego actualizamos los repositorios.

![1768587384395](image/Zimablade2/1768587384395.png)

Y los upgradeamos.

![1768587421477](image/Zimablade2/1768587421477.png)

Una vez actualizado el servidor añadimos el equipos a tailscale que es una VPN gratuita que nos permite acceder a los equipos y que los equipos se vean entre si aunque no esten en la misma red interna. En este caso no seria necesario ya que tengo los equipos en la misma red local, pero esto añade disponibilidad a nuestro servidor y facilidades de añadir proximos dispositivos a la red y poder acceder a ellos desde fuera de la red local, sin tener que configurar proxys. En nuestro caso nos conectamos a la shell de nuestro servidor pve y ejecutamos los siguientes comandos.

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

### Cluster

En este paso vamos a unir este nodo al cluster para que los dos nodos esten unidos.

Vamos al nodo inicial, Datacenter --- Cluster --- Join Information.

![1768588236260](image/Zimablade2/1768588236260.png)

La copiamos y dentro del mismo menu pero del segundo servidor seleccionamos Datacenter --- Cluster --- Join Cluster

![1768588330549](image/Zimablade2/1768588330549.png)

Aqui pegamos la informacion, y nos pedira la contraseña del nodo principal y con esto ya se uniria al cluster.

![1768588484598](image/Zimablade2/1768588484598.png)

## Backup VM

Ahora vamos a configurar el Backup de nuestra VM de el nodo principal, que es el principal trabajo de este servidor. Para realizar las Backups vamos a crear una VM dedicadada a esto, vamos a usar el sistema operativo Proxmox Backup Server, descargamos la ISO desde su sitio web(https://www.proxmox.com/en/downloads) en este caso la ultima version 4.1.

### Config VM

Antes de nada vamos a crear los discos que vamos a usar en esta VM, en este caso son dos uno que lo vamos a llamar Sistema, este se va a encargar de alojar el sistema operativo PBS y otro que ya viene creado por defecto local-lvm es el encargado de guardar las Copias de seguridad.

![1768650829336](image/Zimablade2/1768650829336.png)

Iniciamos con la configuracion de la VM, creamos la VM en el nodo secundario pve2.

![1768651186633](image/Zimablade2/1768651186633.png)

Ahora seleccionamos la ISO y Importante seleccionar el almacenamiento System y previamente a ver subido la ISO de la siguiente manera.

![1768651278160](image/Zimablade2/1768651278160.png)

Luego ya podemos seleccionarla.

![1768651316023](image/Zimablade2/1768651316023.png)

El sistema lo dejamos por defecto y pasamos al disco que tambien lo vamos a dejar por defecto.

En CPU vamos a aprovechar los dos Cores que tiene este dispositivo, lo demas por defecto.

![1768651402499](image/Zimablade2/1768651402499.png)

La memoria le vamos a poner 7000Mib ya que solo tenemos 8GB.

![1768651437189](image/Zimablade2/1768651437189.png)

La Network la vamos a dejar en Bridge.

![1768651465904](image/Zimablade2/1768651465904.png)

En resumen deberia de quedar asi.

![1768651492798](image/Zimablade2/1768651492798.png)

Una vez creada vamos a añadir el disco encargado de las Backups, yo lo voy a configurar con 1TB ya que mi VM ocupa aproximandamente 600GB.

![1768651935799](image/Zimablade2/1768651935799.png)

### Instalacion PBS

Una vez creada la iniciamos y pasamos a la configuracion del servidor PBS, esta configuracion no la voy a documentar ya que es seguir el mismo proceso que con la instalacion de Proxmox VE, ver mas arriba.

Una vez instalado nos metemos a su sitio web.

```
https://192.168.1.51:8007
```

 Y vamos a empezar con la configuracion, primero tenemos que crear el disco ZFS.

![1768652006627](image/Zimablade2/1768652006627.png)

Con esto ya se nos crea automaticamente nuestro Datastore, esto es lo que vamos a conectar a nuestro Datacenter para hacer los Backups.

![1768652118430](image/Zimablade2/1768652118430.png)

### Conexión Datacenter con PBS

Para conectar nuestro PBS al Datacenter vamos a ir a nuestro Datacenter y vamos a añadir este sistema de almacenamiento, toda la informacaion de conexion se puede obtener desde nuestro    PBS

![1768652417873](image/Zimablade2/1768652417873.png)

Ahora vamos a nuestro Datacenter y completamos los datos de conexion.

![1768652490238](image/Zimablade2/1768652490238.png)

Si te da un error 401 probablemente la contraseña o el usuario estan mal.

Ahora ya podemos hacer nuestro primer Backup para esto vamos a la VM que queremos hacer el Backup y seleccionamos Backup Now, yo en este caso voy a usar el modo parar.

![1768652703900](image/Zimablade2/1768652703900.png)

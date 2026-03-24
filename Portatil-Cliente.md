# PREPARACIÓN DE PORTÁTIL PARA HACER PRUEBAS

## Objetivo

Queremos acondicionar un portátil para que sea versátil y sirva para crear proyectos supervisar nuestro HOMELAB y crear servicios sin que pierda su propósito principal como portátil. Para ello vamos a empezar creando una maquina virtual para poder usar Proxmox y asi este conectado a nuestro nodo local pero sin perder su sistema operativo ni su funcionalidad como portátil.

## Requisitos

- VIRTUALBOX

## Instalación VM

Antes de nada vamos a configurar una IP estática para nuestro portátil en nuestra red local.

![1768052779631](image/PORT/1768052779631.png)

Luego vamos a descargar Tailscale para unir por VPN nuestro portátil con el resto de dispositivos de la red.

``` text
https://tailscale.com/download
```

![1768052921598](image/PORT/1768052921598.png)

Vamos a crear la VM con VIRTUALBOX para esto una vez instalado el programa necesitamos el archivo .iso de instalación de PROXMOX VE, vamos a la pagina oficial de Proxmox y descargamos la ultima version de Proxmox VE.

``` text
https://www.proxmox.com/en/downloads/proxmox-virtual-environment
```

Una vez descargamos la ISO que queremos vamos a crear la VM en VIRTUALBOX con la siguiente configuración, yo decidí usar esta configuración pero tu puedes usar cualquier otra según tu caso.

![1767995935802](image/PORT/1767995935802.png)

Yo esta maquina solo la quiero para hacer pruebas no para correr servicios 24/7 por lo que la voy a hacer con recursos limitados ya que no me interesa que me consuma muchos recursos de mi dispositivo.

![1767996120330](image/PORT/1767996120330.png)

El resumen de nuestra instalación debería de ser algo asi, una vez asi la creamos.

![1767996139436](image/PORT/1767996139436.png)

Una vez creada es importante activar la Característica "Nested VT-x/AMD-V" esto activa la virtualización dentro de la VM.

![1767998231260](image/PORT/1767998231260.png)

Configuramos la red de la VM en adaptador puente ya que asi no tenemos que configurar y usa la misma red del Portátil.

![1767997839639](image/PORT/1767997839639.png)

## Configuración

Arrancamos y instalamos Proxmox.

> ***Si te da error al intentar la VM hay que activar la virtualización en la BIOS***

![1767997269265](image/PORT/1767997269265.png)

*Seguimos la configuración de instalación configuramos IP y nombre, **IMPORTANTE** que no repites ni la IP ni el nombre con otros dispositivos.*

> Una vez iniciado nos dará una url que podremos usar para conectarnos a nuestro Proxmox VE y ya estaría configurado.

### Unimos el Proxmox al nodo de nuestro HOMELAB (Deprecated)

Desde las opciones del DataCenter de nuestro HOMELAB en cluster copiamos la información de union.

![1768000566691](image/PORT/1768000566691.png)

Luego hacemos lo mismo pero en el dispositivo nuevo y seleccionamos la opción Join cluster aquí pegamos la información de union de nuestro HOMELAB, y con esto ya tenemos los dos dispositivos en un mismo nodo.

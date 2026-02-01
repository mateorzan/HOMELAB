# PREPARACION DE PORTATIL PARA HACER PRUEBAS

## Objetivo

Queremos acondicionar un portatil para que sea versatil y sirva para crear proyectos supervisar nuestro HOMELAB y crear servicios sin que pierda su proposito principal como portatil. Para ello vamos a empezar creando una maquina virtual para poder usar proxmox y asi este conectado a nuestro nodo local pero sin perder su sistema operativo ni su funcionalidad como portatil.

## Requisitos

- VIRTUALBOX

## Instalacaión VM

Antes de nada vamos a configurar una IP estatica para nuestro portatil en nuestra red local.

![1768052779631](image/PORT/1768052779631.png)

Luego vamos a descargar Tailsacale para unir por VPN nuestro portatil con el resto de dispositivos de la red.

```
https://tailscale.com/download
```

    ![1768052921598](image/PORT/1768052921598.png)

Vamos a crear la VM con VIRTUALBOX para esto una vez instalado el programa necesitamos el archivo .iso de instalacion de PROXMOX VE, vamos a la pagina oficial de Proxmox y descargamos la ultima version de Proxmox VE.

```
https://www.proxmox.com/en/downloads/proxmox-virtual-environment
```

Una vez descargamos la ISO que queremos vamos a crear la VM en VIRTUALBOX con la siguiente configuracion, yo decidi usar esta configuracion pero tu puedes usar cualquier otra segun tu caso.

![1767995935802](image/PORT/1767995935802.png)

Yo esta maquina solo la quiero para hacer pruebas no para correr servicios 24/7 por lo que la voy a hacer con recursos limitados ya que no me interesa que me consuma muchos recursos de mi dispositivo.

![1767996120330](image/PORT/1767996120330.png)

El resumen de nuestra instalacion deberia de ser algo asi, una vez asi la creamos.

![1767996139436](image/PORT/1767996139436.png)

Una vez creada es importante activar la Caracteristica "Nested VT-x/AMD-V" esto activa la virtualizacion dentro de la VM.

![1767998231260](image/PORT/1767998231260.png)

Configuramos la red de la VM en adaptador puente ya que asi no tenemos que configurar y usa la misma red del Portatil.

![1767997839639](image/PORT/1767997839639.png)

## Configuración

Arrancamos y instalamos proxmox.

> ***Si te da error al intentar la VM hay que activar la virtualizacion en la BIOS***

![1767997269265](image/PORT/1767997269265.png)

*Seguimos la configuracion de instalacion configuramos IP y nombre, **IMPORTANTE** que no repites ni la IP ni el nombre con otros dispositivos.*

> Una vez iniciado nos dara una url que podremos usar para conectarnos a nuestro proxmox VE y ya estaria configurado.

### Unimos el proxmox al nodo de nuestro HOMELAB

Desde las opciones del Datacenter de nuestro HOMELAB en cluester copiamos la informacion de union.

![1768000566691](image/PORT/1768000566691.png)

Luego hacemos lo mismo pero en el dispositivo nuevo y seleccionamos la opcion Join cluster aqui pegamos la informacion de union de nuestro HOMELAB, y con esto ya tenemos los dos dispositivos en un mismo nodo.

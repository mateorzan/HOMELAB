# Configuración Raspberry Pi 5

Este equipo lo vamos a centrar en IA, no es un equipo muy potente para esta tarea pero nos sirve para hacer pruebas. Este dispositivo presenta un problema ya que por su arquitectura ARM no es compatible con Proxmox VE por que lo no podremos instalar este sistema operativo y unirlo a nuestro nodo, pero aun asi podemos hacer y probar cosas y aplicarlas en nuestro nodo.

## Requisitos

- Raspberry Pi
- USB/Disco externo.
- Raspberry Imager
- Teclado

## Instalación

Vamos a empezar con la instalacion, en este caso vamos a usar el software Raspberry Pi Imager para crear nuestra disco de arranque.

```
https://www.raspberrypi.com/software/
```

Una vez instalado vamos a conectar nuestro disco de arranque en mi caso un disco SSD externo en el que vamos a instalar el sistema operativo.

Iniciasmos el software y vamos a instalar Raspberry Pi OS Lite en nuestro disco.

![1769961713171](image/Raspberry_pi5/1769961713171.png)

Seguimos los pasos de instalacion que nos indican el software, añadimos un nombre al equipo y usuario que queramos para iniciar sersion. En mi caso no voy a configurar WI-FI ya que lo voy a conectar por cable Ethernet pero algo importante que si hay que seleccionar es la opcion "Activar SSH".

![1769961920574](image/Raspberry_pi5/1769961920574.png)

Con todo esto ya podemos escribir en el disco.

> IMPORTANTE ESTO BORRARA TODO LO QUE TENGAS ALMACENADO EN EL DISCO SELECCIONADO.

Una vez termine la des escribir en el disco ya tenemos todo listo para empezar con la instalacion del sistema operativo.

Conectamos el disco externo a uno de los USB de nuestra Raspberry Pi y la iniciamos.

Una vez iniciado vamos a la pagina web local de nuestro Router para ver la IP local de nuestra Raspberry Pi y asi poder conectarnos por SSH.

```
http://192.168.1.1/
```

Una vez sabemos la IP nos conectamos con el nombre de usuario (el que configuraste en Raspberry Pi Imager) y la IP local.

```
ssh mateorzan@192.168.1.44
```

Con todo esto ya tenemos todo instalado ahora vamos a pasar con la configuración.

## Configuración

El primer paso que vamos a hacer es configurar un IP estatica, ejecutamos en la terminal el siguiente comando.

```
sudo nmtui
```

Dentro editamos la conexion y escribimos la IP local que tengamos libre, Importante que ningun dispositivo de la red local tenga esa IP pillada.

![1769963137805](image/Raspberry_pi5/1769963137805.png)

Con esto ya tenemos la IP estatica configurada.

Lo siguiente que vamos a configurar va a ser Tailscale para poder acceder al dispositivo desde fuera de la red local, para ello desde la propia web de Tailscale seleccionamos para añadir un cliente Linux y nos dara el comando de instalacion.

```
curl -fsSL https://tailscale.com/install.sh | sh
```

Una vez instale nos mandara ejecutar otro comando

```
sudo tailscale up
```

Este comando nos dara una URL a la cual tenemos que acceder para aceptar el dispositivo en nuestra red de Tailscale.

Con esto ya tenemos tailscale instalado y funcionando.

## Servicios

### OLLAMA

Vamos a probar a correr un modelo de IA local para ver como rinde.

```
curl -fsSL https://ollama.com/install.sh | sh
```

Una vez instalado vamos a correr un modelo ligero, vamos a probar con LFM2.5.

```
ollama run lfm2.5-thinking
```

Una vez instalado el modelo y que vemos que funciona bien vamos a instalar un chat para poder usar el modelo comodamente, en mi caso elegi Open-webui(https://github.com/open-webui/open-webui).

Para usar este chat necesitamos tener Docker instalado.

```
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

```
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Con este comando comprobamos que se instalo bien.

```
sudo docker run hello-world
```

Tambien vamos a comprobar que tengamos Python instalado.

```
sudo apt install python3
sudo apt install python3-venv python3-pip
```

Una vez instalado todo ejecutamos el siguiente comando para ejecutar el contenedor docker.

```
span
```

Con el comando `sudo docker ps` podemos ver como esta el contenedor, si esta healthy podemos acceder a el con la IP de la maquina y el puerto 3000

```
http://192.168.1.52:8080
```

### OpenClaw

Instalamos OpenClaw con el siguiente comando.

```
curl -fsSL https://openclaw.ai/install.sh | bash
```

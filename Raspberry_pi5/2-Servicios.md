## Servicios

### OLLAMA

Vamos a probar a correr un modelo de IA local para ver como rinde.

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Una vez instalado vamos a correr un modelo ligero, vamos a probar con LFM2.5.

```bash
ollama run lfm2.5-thinking
```

Una vez instalado el modelo y que vemos que funciona bien vamos a instalar un chat para poder usar el modelo cómodamente, en mi caso elegí [Open-webui](https://github.com/open-webui/open-webui).

Para usar este chat necesitamos tener Docker instalado.

```bash
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

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Con este comando comprobamos que se instalo bien.

```bash
sudo docker run hello-world
```

También vamos a comprobar que tengamos Python instalado.

```bash
sudo apt install python3
sudo apt install python3-venv python3-pip
```

Una vez instalado todo ejecutamos el siguiente comando para ejecutar el contenedor docker.

```bash
span
```

Con el comando `sudo docker ps` podemos ver como esta el contenedor, si esta healthy podemos acceder a el con la IP de la maquina y el puerto 3000

```text
http://192.168.1.52:8080
```

### OpenClaw

Instalamos OpenClaw con el siguiente comando.

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Una vez instalado configuramos y añadimos un proveedor de IA en mi caso estoy usando codex, no voy a explicar como hice esto ya que es algo que me puede comprometer, pero actualmente tengo codex conectado a mi OpenClaw y me comunico con el a traves de un bot de telegram.

Con este servicio actualmente me encuentro haciendo pruebas pero no tengo nada corriendo lo uso mas como un asistente ya que ahora mismo uso la raspberry como herramienta de monitorización de el resto de mis maquinas virtuales y servicios.

### Jellyfin

Vamos a montar un Jellyfin en este servidor, con esto quiero ver el rendimiento de este servicio en una raspberry Pi 5. Actualmente tengo este servicio corriendo en mi ZimaOS, debido a la carga de otros servicios no funciona todo lo bien que esperaria.

Para montar este jellyfin vamos a aprovechar la estructura que ya tengo montada en mi ZimaOS y vamos a crear un almacenamiento compartido entre mi raspberry Pi 5 y mi ZimaOS asi solo tengo que recrear mi servidor Jellyfin en este servidor.

#### Requisitos

- Docker
- Docker Compose
- smdbclient y cifs-utils

#### Docker Compose

Para crear nuestro servidor jellyfin vamos a usar el siguiente compose.yml

```Shell
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    # Optional - specify the uid and gid you would like Jellyfin to use instead of root
    user: uid:gid
    ports:
      - 8096:8096/tcp
      - 7359:7359/udp
    volumes:
      - /path/to/config:/config
      - /path/to/cache:/cache
      - type: bind
        source: /path/to/media
        target: /media
      - type: bind
        source: /path/to/media2
        target: /media2
        read_only: true
      # Optional - extra fonts to be used during transcoding with subtitle burn-in
      - type: bind
        source: /path/to/fonts
        target: /usr/local/share/fonts/custom
        read_only: true
    restart: 'unless-stopped'
    # Optional - alternative address used for autodiscovery
    environment:
      - JELLYFIN_PublishedServerUrl=http://example.com
    # Optional - may be necessary for docker healthcheck to pass if running in host network mode
    extra_hosts:
      - 'host.docker.internal:host-gateway'
```

Una vez creado y levantado ya podemos acceder al servidor desde nuestro navegador con la url

`htpp://IP:8096`

#### SMDB

Ahora como explique antes vamos a aprovechar las peliculas que ya tengo en mi servidor y vamos a montar la carpeta compartida por SMDB en nuestra Raspberry.

La ruta es la siguiente que compartimos en nuestro servidor es:

`//zimaos/media`

Ahora para poder acceder a esta ruta tenemos que instalar primero las herramientas para poder acceder al smbd, lo hacemos con estos comandos.

`sudo apt update`

`sudo apt install samba-client cifs-utils -y`

Ahora para montar esta nueva ruta de almacenamiento usamos el siguiente comando.

`sudo mount -t cifs //IP_SERVIDOR/nombre_carpeta /home/mateorzan/media -o username=tu_usuario,password=tu_contraseña,uid=1000,gid=1000`

Ejemplo

`sudo mount -t cifs //zimaos/media /home/mateorzan/media -o username=****,password=******,uid=1000,gid=1000`

Como lo estamos montando con nuestro usuario, para que se monte automaticamente siempre al arrancar necesitamos crear un archivo que guarde las credenciales.

`sudo nano /etc/samba/credenciales`

Luego protegemos el archivo.

`sudo chmod 600 /etc/samba/credenciales`

Luego creamos el archivo que hace que se monte la ruta siempre.

`sudo nano /etc/fstab`

`//IP_SERVIDOR/nombre_carpeta  /home/mateorzan/media  cifs  credentials=/etc/samba/credenciales,uid=1000,gid=1000,_netdev  0  0`

Por ultimo probamos que todo funciona bien y no nos da ningun error de sintaxis.

`sudo mount -a`

### Homelab Nexus

Cree mi imagen personalizada de un dashboard tipo Homepage pero que integras las Apis de Uptime-Kuma, Beszel y Gotify para asi de una vista ver toda la informacion de esos tres servicios en uno, le añadi links a los monitores para que puedas configurarlo y que haga el mismo uso que le doy a Homepage agragando que tengo un monitoreo avanzado y superior a Homepage. Sus funcionalidades estan explicadas en el repo pero tiene todo lo que necesito ahora mismo, monitorizacion y links para acceder a los servicios o servidores que necesito, ademas de bookmarks que puedo personalizar como quiera.

#### Docker Compose

```Shell
services:
  homelab-nexus:
    image: ghcr.io/mateorzan/homelab-nexus:latest
    container_name: homelab-nexus
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - PORT=3000
      - HOST=0.0.0.0
      - POLL_INTERVAL_MS=1000
      - CACHE_TTL_MS=1000
      - BG_IMAGE=
      - BESZEL_URL=http://tu-beszel:8090
      - BESZEL_USER_EMAIL=tu@email.com
      - BESZEL_USER_PASSWORD=tu-password
      - UPTIMEKUMA_URL=http://tu-uptimekuma:3001
      - UPTIMEKUMA_STATUS_PAGE=tu-status-page
      - GOTIFY_URL=http://tu-gotify:8081
      - GOTIFY_TOKEN=tu-token
    volumes:
      - homelab-nexus-data:/app/data

volumes:
  homelab-nexus-data:
```

Funciona como cualquier compose, pero como por ahora la imagen es privada ya que estoy en testing hay que hacer login con

```
# LOGIN
echo "TU_GITHUB_TOKEN" | docker login ghcr.io -u mateorzan --password-stdin

# ARRANQUE
docker-compose up -d
```

Con esto nos metemos a la url en el puerto :3000 y ya lo tenemos.

![1788525588962](image/Raspberry_pi5/1788525588962.png)

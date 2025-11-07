# HOMELAB PROJECT CON RASPBERRY Y ZIMABLADE SERVERS

En este repositorio voy a explicar como yo configure mi actual homelab, compartire algunos archivos de configuracion que pueden ser de ayuda para la gente que quiere hacer algo parecido en su propio servidor.

## Historico
### Estructura actual :

Una raspberry pi con CasaOS instalado localmente con diferentes servicios creados a traves de CasaOS

NextCloud, servicio Cloud
Jellyfin, servicio de medios
Nginx Proxy Manager, proxy que publica los servicios
Prowlar
Sonar, servicio de series
Radarr, servicio de peliculas
DdnsUpdater, servicio para actualizar la ip publica
Sure, servicio finanzas

### Situacion actual :

Dispongo de dos servidores mas, dos zimablades, queremos ampliar nuestro Homelab con dos servidores mas, para esto vamos a implementar un Swarm de docker. En este Swarm vamos a tener tres nodos, un nodo manager(raspberrypi) y dos workers(zimablades).

La idea de esta estructura es crear un red de alta disponibilidad y respaldada y gestionar cargas de trabajo entre nodos, de todo esto se encarga swarm.


### Estructura final(Objetivo) :

Crearemos todos los servicios con swarm para que compartan la carga entre los diferentes nodos. La intencion es ir migrando poco a poco todos los servicios de la raspberry que se ejecutan con docker pasarlo a docker swarm.

## Problemas o inquietudes :

Como la raspberry y zima tienen diferentes arquitecturas no todos los servicios se pueden ejecutar en los dos servidores, por lo que hay servicios que no son multi-arch que hay que ejecutar por separado. 

Tuve problemas para montar el servicio nfs como un servicio en docker swarm fíjate que todas las rutas, la els y imágenes estén bn elegidas y sean compatibles.

## Soluciones y Progresos :

Primero antes de hacer nada añadimos los equipos a tailscale que es una VPN gratuita que nos permite acceder a los equipos y que los equipos se vean entre si aunque no esten en la misma red interna. En este caso no seria necesario ya que tengo los equipos en la misma red local, pero esto añade disponibilidad a nuestro servidor y facilidades de añadir proximos dispositivos a la red y poder acceder a ellos desde fuera de la red local, sin tener que configurar proxys.

`curl -fsSL https://tailscale.com/install.sh | sh
`sudo tailscale up`

Puedes ver que esta bien configurado con

`tailscale status`
`ping <ip_tailscale_otro_dispositivo>`


Por ahora creamos el swarm, empezamos creando el manager y con el token del manager creamos los workers

`docker swarm init --advertise-addr <ip_tailscale_pi>`
`docker swarm join --token <token> <ip_tailscale_pi>:2377`

Verificamos:

`docker node ls`


Para solucionar el problema de aquitecturas en swarm configuramos unas variables para que los servicios que no sean multi-arch se ejecuten en el servidor correcto.

`docker node update --label-add arch=arm64 --label-add rol=manager raspberrypi`
`docker node update --label-add arch=amd64 --label-add rol=worker zimablade1`
`docker node update --label-add arch=amd64 --label-add rol=worker zimablade2`

Verficamos:

`docker node ls`
`docker node inspect raspberrypi --format '{{.Spec.Labels}}'`

Vamos a crear un servidor NFS compartido para mas adelante crear los volumenes de los servicios. En este caso vamos a crear el servidor nfs en una instancia de docker metida en el swarm.

Creamos el archivo `nfs-stack.yml`

```bash
 # nfs-stack.yml
version: "3.9"
services:
  nfs-server:
    image: octopusdeploy/nfs-server
    privileged: true
    restart: unless-stopped
    volumes:
      - /mnt/ssd/nfs:/nfsshare
    environment:
      - SHARED_DIRECTORY=/nfsshare
    ports:
      - target: 2049
        published: 2049
        mode: host
        protocol: tcp
      - target: 2049
        published: 2049
        mode: host
        protocol: udp
      - target: 111
        published: 111
        mode: host
        protocol: tcp
      - target: 111
        published: 111
        mode: host
        protocol: udp
    networks:
      - nfs_net
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.arch == arm64]   # ← Se mueve entre managers
      restart_policy:
        condition: on-failure
networks:
  nfs_net:
    driver: overlay
```

Lo lanzamos como un servicio de docker swarm

`docker stack deploy -c nfs-stack.yml nfs`

Comprobamos:

```bash
docker service ls
docker service ps nfs_nfs-server
```

Una vez creado montamos la carpeta o carpetas en el otro servidor.

`sudo mount -t nfs 192.168.1.100:/mnt/ssd/nfs /mnt/nfs/glace_d`

Creamos el Dashboard que vamos a utilizar en este caso Glance.

`mkdir glance && cd glance && curl -sL https://github.com/glanceapp/docker-compose-template/archive/refs/heads/main.tar.gz | tar -xzf - --strip-components 2`

Luego, edita los siguientes ficheros a tu gusto:

`docker-compose.yml` to configure the port, volumes and other containery things
`config/home.yml to` configure the widgets or layout of the home page
`config/glance.yml` if you want to change the theme or add more pages

Editamos el archivo docker-compose.yml para que corresponda con nuestra estructura

```bash
version: "3.9"

services:
  glance:
    image: glanceapp/glance:latest
    restart: unless-stopped
    volumes:
      - ./config:/app/config
      - ./assets:/app/assets
      - /etc/localtime:/etc/localtime:ro
      # Para ver los contenedores del clúster:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "7000:8080"
    environment:
      - TZ=Europe/Madrid
      # Carga variables del .env manualmente si las necesitas, ejemplo:
      # - GLANCE_AUTH_KEY=${GLANCE_AUTH_KEY}
    networks:
      - glance_net
    deploy:
      mode: replicated
      replicas: 2
      restart_policy:
        condition: any
      update_config:
        parallelism: 1
        delay: 10s
      #placement:
        # constraints:
          # - node.role == manager  # opcional: Glance suele necesitar acceso al socket Docker local

networks:
  glance_net:
    driver: overlay
```
En esta estructura para que el docker compose se pueda lanzar necesitamos darle permisos de manager al zimablade

`sudo docker node promote zimablade1`

Cuando este listo, ejecuta:

`docker stack deploy -c docker-compose2.yml glance`

Una vez lanzado convertimos el Zima a worker otra vez:

`sudo docker node demote zimablade1`

El servidor deberia estar disponible en:

`http://<IP_zimablade1>:7070`

Creamos nuestro primer servicio, en este caso Portrainer

`docker volume create portainer_data`
`docker run -d -p 9000:9000 -p 8000:8000 \`
  `--name portainer --restart=always \`
  `-v /var/run/docker.sock:/var/run/docker.sock \`
  `-v portainer_data:/data \`
  `portainer/portainer-ce` 

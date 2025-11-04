Estructura actual:

Una raspberry pi con CasaOS instalado localmente con diferentes servicios creados a traves de CasaOS

NextCloud, servicio Cloud
Jellyfin, servicio de medios
Nginx Proxy Manager, proxy que publica los servicios
Prowlar
Sonar, servicio de series
Radarr, servicio de peliculas
DdnsUpdater, servicio para actualizar la ip publica
Sure, servicio finanzas

Situacion actual:

Dispongo de dos servidores mas, dos zimablades, queremos ampliar nuestro Homelab con dos servidores mas, para esto vamos a implementar un Swarm de docker. En este Swarm vamos a tener tres nodos, un nodo manager(raspberrypi) y dos workers(zimablades).

La idea de esta estructura es crear un red de alta disponibilidad y respaldada y gestionar cargas de trabajo entre nodos, de todo esto se encarga swarm.

Estructura final:

Crearemos todos los servicios con swarm para que compartan la carga entre los diferentes nodos. La intencion es ir migrando poco a poco todos los servicios de la raspberry que se ejecutan con docker pasarlo a docker swarm.

Problemas o inquietudes:

Como la raspberry y zima tienen diferesntes arquitecturas no todos los servicios se pueden ejecutar en los dos servidores, por lo que hay servicios que no son multi-arch que hay que ejecutar por separado. 

Soluciones y Progresos :

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

Creamos el Dashboard que vamos a utilizar en este caso Glance.

`mkdir glance && cd glance && curl -sL https://github.com/glanceapp/docker-compose-template/archive/refs/heads/main.tar.gz | tar -xzf - --strip-components 2`

Luego, edita los siguientes ficheros a tu gusto:

`docker-compose.yml` to configure the port, volumes and other containery things
`config/home.yml to` configure the widgets or layout of the home page
`config/glance.yml` if you want to change the theme or add more pages

Cuando este listo, ejecuta:

docker compose up -d

If you encounter any issues, you can check the logs by running:

docker compose logs

El servidor deberia estar disponible en:

`http://<IP_RASPBERRY>:7070`


Creamos nuestro primer servicio, en este caso Portrainer

`docker volume create portainer_data`
`docker run -d -p 9000:9000 -p 8000:8000 \`
  `--name portainer --restart=always \`
  `-v /var/run/docker.sock:/var/run/docker.sock \`
  `-v portainer_data:/data \`
  `portainer/portainer-ce` 

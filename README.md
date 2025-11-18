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

Crearemos todos los servicios con swarm para que compartan la carga entre los diferentes nodos. La intencion es ir migrando poco a poco todos los servicios de la raspberry que se ejecutan con docker.

## Problemas o inquietudes :

Como la raspberry y zima tienen diferentes arquitecturas no todos los servicios se pueden ejecutar en los dos servidores, por lo que hay servicios que no son multi-arch que hay que ejecutar por separado. 

Tuve problemas para montar el servicio nfs como un servicio en docker swarm fíjate que todas las rutas, la els y imágenes estén bn elegidas y sean compatibles.

Debido a los problemas y poco soporte viste en docker swarm vamos a empezar de 0 y probar a instalar K3s, el cual es una distribucion mas ligera de Kubernetes pensada para ahorrar memoria y recursos del sistema, lo cual es justo lo que necesitamos.

## Soluciones y Progresos :

Instalamos K3s a traves de su pagina oficial (https://docs.k3s.io/quick-start) seguimos la guia.

`curl -sfL https://get.k3s.io | sh -`


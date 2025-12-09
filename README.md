# HOMELAB PROJECT CON RASPBERRY Y ZIMABLADE SERVERS

En este repositorio voy a explicar como yo configure mi actual homelab, compartire algunos archivos de configuracion que pueden ser de ayuda para la gente que quiere hacer algo parecido en su propio servidor.

## Histórico

### Estructura actual :

Una raspberry pi con CasaOS instalado localmente con diferentes servicios creados a traves de CasaOS

NextCloud, servicio Cloud
Jellyfin, servicio de medios Multimedia
Nginx Proxy Manager, proxy que publica los servicios
Prowlar, indexer
Sonar, servicio de series
Radarr, servicio de peliculas
DdnsUpdater, servicio para actualizar la ip publica
Sure, servicio finanzas

### Situacion actual :

Dispongo de dos servidores más, dos zimablades, queremos ampliar nuestro Homelab con dos servidores más, los cuales queremos poner tambien a funcionar y crear un nodo con tres servidores que corran diferentes servicios. Estos zimablades traen ya preinstalado CasaOS ya que son de los creadores de este sistema.

La idea de esta estructura es crear un red de alta disponibilidad, respaldada y gestionar cargas de trabajo entre nodos, y disponer de los servicios 24/7.

### Estructura final(Objetivo) :

La intencion final de este proyecto es crear una red de nodos de alta disponibilidad y respaldada,este proyecto es muy ambicioso y aun no se el alcance de este mismo, ire actualizando con las novedades este Readme.

### Ideas y pruebas que voy realizando:

#### Estructural

| APLICADO | PROBADO | TECNOLOGÍA | NOTAS                                                                                                                                                                                                                                         |
| :------: | :-----: | :----------: | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|    NO    |   SI   | Docker Swarm | No tiene soporte y es bastante limitado tengo que probar con otro<br />orquestador.                                                                                                                                                           |
|    SI    |   SI   |  Tailscale  | Es una VPN que conecta todos los servidores entre si por una red<br />publica pero privada, hace accesible a todos los servidores desde <br />cualquier sitio si estas conectado a esta VPN.                                                  |
|    NO    |   NO   |   Proxmox   | Es una plataforma de vistualizacion de servidores, tiene<br />una cierta orquestación ya que te permite gestionar nodos.<br />Esto seria una buena opción para mi caso, queda pendiente<br />necesito los discos duros para los zimablades. |
|    NO    |   NO   |  Kubernetes  | Es el orquestador mas usado a nivel de servicios queda<br />pendiente de ver como funciona y si tiene sentido en <br />mi estructura.                                                                                                        |
|    SI    |   SI   |    CasaOS    | Todos mis servidores tienen este sistema instalado<br />facilita mucho los crear servicios y implementarlos.                                                                                                                                  |

#### Servicios

| APLICADO | APROBADO |      SERVICIO      | NOTAS                                                                              |
| :------: | :------: | :-----------------: | ---------------------------------------------------------------------------------- |
|    SI    |    SI    |      NextCloud      | Almacenamiento Cloud Local.                                                        |
|    SI    |    SI    |      Jellyfin      | Servicio Multimedia                                                                |
|    SI    |    SI    |        Sure        | Servicio de gestion de gastos                                                      |
|    SI    |    SI    |       Deluge       | Servicio Torrent                                                                   |
|    SI    |    SI    |      Downtify      | Servicio Musica                                                                    |
|    SI    |    SI    |        Sonar        | Servicio busqueda de Musica                                                        |
|    SI    |    SI    |       Radarr       | Servicio busqueda de Pelis                                                         |
|    SI    |    SI    | Nginx Proxy Manager | Servicio Proxy inverso para exponer servicios<br />con certificados SSL.           |
|    SI    |    SI    |       Prowlar       | Servicio Indexer Torrent                                                           |
|    SI    |    SI    |    Ddns-Updater    | Serviucio que actualiza la ip publica de<br />nuestro router, conectado a duckdns. |
|    NO    |    NO    |       Asenble       |                                                                                    |
|    NO    |    NO    |       Coolify       |                                                                                    |

## Problemas o inquietudes :

Como la raspberry y zima tienen diferentes arquitecturas no todos los servicios se pueden ejecutar en los dos servidores, por lo que hay servicios que no son multi-arch que hay que ejecutar por separado.

Tuve problemas para montar el servicio nfs como un servicio en docker swarm fíjate que todas las rutas, la els y imágenes estén bn elegidas y sean compatibles.

Debido a los problemas y poco soporte viste en docker swarm vamos a empezar de 0 y probar a instalar K3s, el cual es una distribucion mas ligera de Kubernetes pensada para ahorrar memoria y recursos del sistema, lo cual es justo lo que necesitamos.

## TroubleShooting

### CasaOS

#### No funciona URL Web

Reinstalar CasaOS

```
curl -fsSL https://get.casaos.io/uninstall | bash
```

Borrar CasaOS

```
curl -fsSL https://get.casaos.io | bash
```

Instalar CasaOS

```
reboot
```

Reiniciar.

*Si esto sigue sin funcionar y crees que es un problema de puertos.*

```
cd /etc/casaos
```

```
nano gateway.ini
```

*Entramos a la configuracion del puerto y lo cambiamos por uno disponible, ej: port=90.*

```
sudo systemctl restart casaos-gateway
```

## Soluciones y Progresos :

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

Primero antes de hacer nada añadimos los equipos a tailscale que es una VPN gratuita que nos permite acceder a los equipos y que los equipos se vean entre si aunque no esten en la misma red interna. En este caso no seria necesario ya que tengo los equipos en la misma red local, pero esto añade disponibilidad a nuestro servidor y facilidades de añadir proximos dispositivos a la red y poder acceder a ellos desde fuera de la red local, sin tener que configurar proxys.

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

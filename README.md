# HOMELAB PROJECT CON RASPBERRY Y ZIMABLADE SERVERS

En este repositorio voy a explicar como yo configure mi actual homelab, compartire algunos archivos de configuracion que pueden ser de ayuda para la gente que quiere hacer algo parecido en su propio servidor.

## Estructura del repositorio.

Por ahora solo tenemos la documentacion que explica la configuracion de los diferentes dispositivos y a mayores un documento de Troubleshooting que contiene los errores que me fueron surgiendo y como los consegui solucionar.

- Portatil-Cliente.md, configuracion Portatil.
- Readme.md, Resumen contexto y registro.
- Raspberry_pi5.md, configuracion Raspberry Pi5.
- Troubleshooting.md, solución de errores varios.
- Zimablade1.md, configuracion e instalación Zimablade1.
- Zimablade2.md, configuracion e instalación Zimablade2.

## Histórico

### Estructura Inicial :

Una raspberry Pi5 con CasaOS instalado localmente con diferentes servicios creados a traves de CasaOS

NextCloud, servicio Cloud
Jellyfin, servicio de medios Multimedia
Nginx Proxy Manager, proxy que publica los servicios
Prowlar, indexer
Sonar, servicio de series
Radarr, servicio de peliculas
DdnsUpdater, servicio para actualizar la ip publica
Sure, servicio finanzas

### Estructura Actual :

Datacenter(proxmox)
  pve, zimablade1 nodo1
  pve2, zimablade2 nodo2/Backup Replicacion.
Raspberry Pi5 PARADO ACTUALMENTE *servidor independiente corriendo servicios de IA*

<img width="464" height="305" alt="image" src="https://github.com/user-attachments/assets/7c9ffe26-e74b-4516-89f0-8c86da11d884" />

### Situacion Objetivo :

Dispongo de dos servidores más, dos zimablades, queremos ampliar nuestro Homelab con dos servidores más, los cuales queremos poner tambien a funcionar y crear un nodo con tres servidores que corran diferentes servicios. Estos zimablades traen ya preinstalado CasaOS ya que son de los creadores de este sistema.

La idea de esta estructura es crear un red de alta disponibilidad, respaldada y gestionar cargas de trabajo entre nodos, y disponer de los servicios 24/7. Para conseguir esto vamos a instalar Proxmox VE y crear un nodo con repliacion y backups Diarias para asegurar esta alta disponibilidad y respaldo.

Tambien vamos a utilizar la Rapsberry Pi5 sobrante para proyectos centrados en IA ya que esta no es compatible con Proxmox por su arquitectura ARM.

### Estructura final(Objetivo) :

La intencion final de este proyecto es crear una red de nodos de alta disponibilidad y respaldada,este proyecto es muy ambicioso y aun no se el alcance de este mismo, probablemente ira a mas, ire actualizando con las novedades este Readme.

La estructura final tendria que ser la siguiente:

Datacenter(proxmox)
  pve, zimablade1 nodo1
  pve2, zimablade2 nodo2/Backup Replicacion.
Raspberry Pi5 *servidor independiente corriendo servicios de IA*

<img width="464" height="305" alt="image" src="https://github.com/user-attachments/assets/ba976872-2546-44f5-8718-6d2502ca63ec" />

### Ideas y pruebas que voy realizando:

#### Estructural

| APLICADO | PROBADO | TECNOLOGÍA | NOTAS                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| :------: | :-----: | :----------: | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|    NO    |   SI   | Docker Swarm | No tiene soporte y es bastante limitado tengo que probar con otro<br />orquestador.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|    SI    |   SI   |  Tailscale  | Es una VPN que conecta todos los servidores entre si por una red<br />publica pero privada, hace accesible a todos los servidores desde <br />cualquier sitio si estas conectado a esta VPN.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|    SI    |   SI   |   Proxmox   | Es una plataforma de vistualizacion de servidores, tiene<br />una cierta orquestación ya que te permite gestionar nodos.<br />Esto seria una buena opción para mi caso, queda pendiente<br />necesito los discos duros para los zimablades.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
|    NO    |   NO   |  Kubernetes  | Es el orquestador mas usado a nivel de servicios queda<br />pendiente de ver como funciona y si tiene sentido en <br />mi estructura.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
|    SI    |   SI   |    CasaOS    | Todos mis servidores tienen este sistema instalado<br />facilita mucho los crear servicios y implementarlos.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|    SI    |   SI   |  Proxmox BS  | Proxmox Backup Server (PBS) es una solución de copias de seguridad<br />empresarial, gratuita y de código abierto, diseñada específicamente para <br />proteger entornos virtualizados basados en Proxmox VE. Permite realizar <br />backups incrementales, rápidos y eficientes de máquinas virtuales, contenedores <br />y hosts, usando técnicas como deduplicación, compresión y cifrado, lo que <br />reduce mucho el espacio en disco y el tráfico de red. PBS se gestiona desde <br />una interfaz web sencilla.                                                                                                                                                                                                                                     |
|    SI    |   SI   |  Proxmox VE  | Proxmox VE (Virtual Environment) es una plataforma**open-source de virtualización** que permite gestionar <br />**máquinas virtuales y contenedores** desde una única interfaz web, combinando KVM para VMs y LXC <br />para contenedores. Está basada en **Debian Linux** y está pensada para servidores y homelabs, ofreciendo <br />funciones avanzadas como  **clusters** ,  **alta disponibilidad** ,  **backups** ,  **replicación** , **snapshots** y gestión <br />flexible del almacenamiento (ZFS, LVM, Ceph, NFS, etc.), todo sin coste de licencia obligatorio. Es muy popular <br />tanto en entornos profesionales como domésticos por su  **potencia, estabilidad y facilidad de uso** . |

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
|    NO    |    NO    |     Frigate NVR     |                                                                                    |

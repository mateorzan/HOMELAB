
# Set Up Zimablade2 servidor secundario/BACKUP 👮

## Objetivo

Usaremos uno de los Zimablades como servidor secundario, en este servidor vamos a ejecutar las copias de seguridad diarias para asegurar que no se pierda información, ademas también replicaremos la VM por si el servidor principal cae este lo respalde.

## Requisitos

* Zimablade con RAM 8gb
* Pincho USB 8gb
* Cable Ethernet
* Monitor y adaptador miniDP a DP
* HUB usb

## Preparación Instalación

### Software

Vamos a empezar con la instalación, lo primero que necesitamos es desde otro dispositivo es descargar la iso de nuestro nuevo sistema operativo, Proxmox. Para ello vamos a la pagina oficial de Proxmox([https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)), en ella entramos en el apartado de downloads, aquí tenemos los diferentes sistemas que hay disponibles en este caso vamos a usar Proxmox Virtual environment, lo seleccionamos y dentro de el veremos las diferentes versiones, este proyecto fue creado con la ultima version actual, Proxmox VE 8.4 ISO.

### Disco de arranque

Una vez instalado nuestro archivo de instalación ISO necesitamos crear el dispositivo de arranque con el que realizaremos la instalación, para esto necesitamos una unidad de almacenamiento, por ejemplo, un USB. En este proyecto se uso un usb cualquiera de 8 gb, puedes usar cualquiera unidad extraible que supere el tamaño de la ISO.

Para poder crear nuestro dispositivo de arranque necesitamos un programa, en este caso utilizamos Rufus, lo puedes descargar en su pagina oficial([https://rufus.ie/es/#download](https://rufus.ie/es/#download)).

## Proceso de instalación

### Proxmox

Seguimos el tutorial creado por los propios creadores de zimablade ([ZimaBlade_proxmoxInstall](https://www.zimaspace.com/docs/es/zimaboard/ZimaBlades-Cluster-PVE-Makes-Your-Service-Migratable))

*El Hub si se inicia la zima con el no funciona hay q iniciar con el y mientras inicia meterlo.*

Creamos el usb de instalación con RUFUS y con esta configuración, es importante que el esquema de la partición sea MBR.

![1766319984358](image/README/1766319984358.png)

### Inicio de instalación

Una vez tenemos los pasos previos completados pasamos a la instalación del software para instalar el sistema operativo es como cualquier otro sistema operativo, arrancamos el servidor conectado a una pantalla, un teclado y con el disco de instalación, por esto es necesario el hub USB.

Mientras nuestro servidor inicia presionamos la teclear F2 asi entraremos en la BIOS, aquí tenemos que editar el orden de arranque y indicar que arranque con el disco de instalación USB o el que usaste para la instalación. Una vez seleccionado pulsamos F10 para salir guardando los cambios, el servidor se reiniciara y arrancara con la instalación.

Es importante que en la BIOS cambiar el Boot Options Priorities y seleccionar el usb como primero ya que sino iniciara con el almacenamiento interno y no instalara. Para entrar en la BIOS pulsamos DEL(Supr).

![1766322140289](image/README/1766322140289.png)

Una vez configurado dentro de la BIOS en el ultimo menu seleccionamos save and exit con esto se nos guardara y se reiniciara solo.

![1766322179110](image/README/1766322179110.png)

Una vez iniciado nos saldrá el menu de instalación en nuestro caso seleccionaremos modo graphical y seguiremos esta configuración. IMPORTANTE NO INSTALAR PROXMOX EN EL DISCO SSD INTERNO HAY QUE INSTALARLO EN EL DISCO HDD EXTERNO.

SI VES QUE CON EL GRÁFICO NO TE FUNCIONA PRUEBA CON EL TERMINAL CON LA MISMA CONFIGURACIÓN.

> *Una vez termina de instalar se nos reiniciara y antes de que se inicie hay que quitar el usb de instalación para que incie con el disco con el que hicimos la instalación.*

Ahora nos pedirá meternos en la web para hacer la instalación inicial.

![1772005781602](image/Zimablade2/1772005781602.png)

Luego de seleccionar el disco seleccionamos la region y nuestra franja horaria.

![1772005826958](image/Zimablade2/1772005826958.png)

Añadimos la cuenta de admin con un correo y una contraseña.

![1772005858892](image/Zimablade2/1772005858892.png)

Por ultimo configuramos el hostname y la red, este paso es muy importante hacerlo bien.

![1772005915807](image/Zimablade2/1772005915807.png)

Si todo esta bien nos quedaría algo asi.

![1772005940294](image/Zimablade2/1772005940294.png)

## Configuración

Una vez instalado Proxmox nos da una URL con la cual tenemos todo el panel de administración de el servidor.

```text
https://192.168.1.49:8006/
```

Aquí iniciaremos sesión con el usuario y contraseña creados durante la instalación, normalmente root.

![1766322669409](image/README/1766322669409.png)

Luego cambiamos el repositorio de Proxmox al No-Subscription, eliminando los siguientes repositorios.

![1768587272196](image/Zimablade2/1768587272196.png)

Y añadimos el No-Subscription.

![1768587323722](image/Zimablade2/1768587323722.png)

Luego actualizamos los repositorios.

![1768587384395](image/Zimablade2/1768587384395.png)

Y los upgradeamos.

![1768587421477](image/Zimablade2/1768587421477.png)

Una vez actualizado el servidor añadimos el equipos a Tailscale que es una VPN gratuita que nos permite acceder a los equipos y que los equipos se vean entre si aunque no estén en la misma red interna. En este caso no seria necesario ya que tengo los equipos en la misma red local, pero esto añade disponibilidad a nuestro servidor y facilidades de añadir próximos dispositivos a la red y poder acceder a ellos desde fuera de la red local, sin tener que configurar Proxys. En nuestro caso nos conectamos a la shell de nuestro servidor pve y ejecutamos los siguientes comandos.

### Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Instalamos Tailscale.

```bash
sudo tailscale up
```

Activamos Tailscale y seguimos las indicaciones para vincularlo.

```bash
tailscale status
```

Puedes ver que esta bien configurado.

### Cluster

En este paso vamos a unir este nodo al cluster para que los dos nodos estén unidos.

Vamos al nodo inicial, DataCenter --- Cluster --- Join Information.

![1768588236260](image/Zimablade2/1768588236260.png)

La copiamos y dentro del mismo menu pero del segundo servidor seleccionamos DataCenter --- Cluster --- Join Cluster

![1768588330549](image/Zimablade2/1768588330549.png)

Aquí pegamos la información, y nos pedirá la contraseña del nodo principal y con esto ya se unirla al cluster.

![1768588484598](image/Zimablade2/1768588484598.png)

# Virtual Machine PBS

## Backup

Ahora vamos a configurar el Backup de nuestra VM de el nodo principal, que es el principal trabajo de este servidor. Para realizar las Backups vamos a crear una VM dedicada a esto, vamos a usar el sistema operativo Proxmox Backup Server, descargamos la ISO desde su sitio web([https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)) en este caso la ultima version 4.1.

### Config VM

Antes de nada vamos a crear los discos que vamos a usar en esta VM, en este caso son dos uno que lo vamos a llamar Sistema, este se va a encargar de alojar el sistema operativo PBS y otro que ya viene creado por defecto local-lvm es el encargado de guardar las Copias de seguridad.

![1768650829336](image/Zimablade2/1768650829336.png)

Iniciamos con la configuración de la VM, creamos la VM en el nodo secundario pve2.

![1768651186633](image/Zimablade2/1768651186633.png)

Ahora seleccionamos la ISO y Importante seleccionar el almacenamiento System y previamente a ver subido la ISO de la siguiente manera.

![1768651278160](image/Zimablade2/1768651278160.png)

Luego ya podemos seleccionarla.

![1768651316023](image/Zimablade2/1768651316023.png)

El sistema lo dejamos por defecto y pasamos al disco que también lo vamos a dejar por defecto.

En CPU vamos a aprovechar los dos Cores que tiene este dispositivo, lo demás por defecto.

![1768651402499](image/Zimablade2/1768651402499.png)

La memoria le vamos a poner 7000Mib ya que solo tenemos 8GB.

![1768651437189](image/Zimablade2/1768651437189.png)

La Network la vamos a dejar en Bridge.

![1768651465904](image/Zimablade2/1768651465904.png)

En resumen debería de quedar asi.

![1768651492798](image/Zimablade2/1768651492798.png)

Una vez creada vamos a añadir el disco encargado de las Backups, yo lo voy a configurar con 1TB ya que mi VM ocupa aproximadamente 600GB.

![1768651935799](image/Zimablade2/1768651935799.png)

### Instalación PBS

Una vez creada la iniciamos y pasamos a la configuración del servidor PBS, esta configuración no la voy a documentar ya que es seguir el mismo proceso que con la instalación de Proxmox VE, ver mas arriba.

Una vez instalado nos metemos a su sitio web.

```text
https://192.168.1.51:8007
```

 Y vamos a empezar con la configuración, primero tenemos que crear el disco ZFS.

![1768652006627](image/Zimablade2/1768652006627.png)

Con esto ya se nos crea automáticamente nuestro Datastore, esto es lo que vamos a conectar a nuestro DataCenter para hacer los Backups.

![1768652118430](image/Zimablade2/1768652118430.png)

### Conexión DataCenter con PBS

Para conectar nuestro PBS al DataCenter vamos a ir a nuestro DataCenter y vamos a añadir este sistema de almacenamiento, toda la información de conexión se puede obtener desde nuestro    PBS

![1768652417873](image/Zimablade2/1768652417873.png)

Ahora vamos a nuestro DataCenter y completamos los datos de conexión.

Si te da un error 401 probablemente la contraseña o el usuario están mal.

Ahora ya podemos hacer nuestro primer Backup para esto vamos a la VM que queremos hacer el Backup y seleccionamos Backup Now, yo en este caso voy a usar el modo parar.

![1768652703900](image/Zimablade2/1768652703900.png)

# KeePass Container LXC

## Config CT

Creamos un CT con la siguiente configuración.

![1771228873525](image/Zimablade2/1771228873525.png)

### Tailscale

Instalamos Tailscale en el CT, esto lo hacemos desde el panel de admin de la web de Tailscale.

### Docker

URL para instalar docker en Alpine Linux

```text
https://voidnull.es/instalacion-de-docker-en-alpinelinux/
```

Para que docker funcionara en el CT tuvimos que pasar la ruta /dev/net/tun

![1771247334792](image/Zimablade2/1771247334792.png)

## VaultWarden

### Docker

Una vez instalado docker, lanzamos el docker run con el servicio Vaultwarden

```bash
docker run -d --name vaultwarden \
  -e DOMAIN="http://192.168.1.53:8000" \
  -v /vw-data/:/data/ \
  --restart unless-stopped \
  -p 8000:80 \
  vaultwarden/server:latest
```

Una vez lanzado nos pide que usemos HTTPS para esto yo use Nginx Proxy Manager que ya lo tenia corriendo en mi segundo servidor por lo que solo tuve que configurar el Proxy.

![1771247514107](image/Zimablade2/1771247514107.png)

Ya con esto accedemos con la URL HTTPs y ya podemos usar vaultwarden sin problemas.

## Portainer

### Docker

Para instalar este servicio simplemente lanzamos un docker run.

```bash
docker run -d \
  -p 9000:9443 \
  -p 8001:8001 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Beszel

### Docker

Para lanzar este servicio de monitorización seguimos los pasos de este GitHub.

```text
https://beszel.dev/guide/getting-started
```

En mi caso voy a lanzarlo con docker run.

```bash
docker volume create beszel_data && \
docker run -d \
  --name beszel \
  --restart=unless-stopped \
  --volume beszel_data:/beszel_data \
  -e APP_URL=http://localhost:8090 \
  -p 8090:8090 \
  henrygd/beszel
```

Luego agregamos el Agente a los servidores que queramos monitorizar, esto lo puedes agregar com un binario o como un contenedor docker.

![1771414254217](image/Zimablade2/1771414254217.png)

### Notificaciones

Vamos a configurar que envié las notificaciones a Gotify, para esto necesitamos una URL HTTPS por lo que vamos a configurar una con Tailscale.

```bash
tailscale serve --bg --https=443 http://127.0.0.1:8081
```

Esto nos dará una URL tipo

```text
https://tu-maquina.ts.net
```

Con esto ya podemos usar esta url para que envié las notificaciones, utilizamos el formato de ejemplo que nos proporciona beszel y lo cambiamos con nuestros datos.

```text
gotify://gotify.example.com:443/AzyoeNS.D4iJLVa/?priority=1
```

![1773929277185](image/Zimablade2/1773929277185.png)

Una vez añadida con nuestra url concreta ya debería de mandar todas las notificaciones.

## Alertas

Para configurar las alertas vamos a usar un Servicio de Chat llamado Gotify, este es compatible y esta implementado en Proxmox por lo que simplemente tendremos que lanzar un docker y conectarlo a nuestro DataCenter.

Docker

```bash
docker run -d \
  --name gotify \
  --restart unless-stopped \
  -p 8081:80 \
  gotify/server
```

Luego podemos acceder al Gotify en `http://IP_DE_TU_SERVIDOR:8081`

*El usuario y contraseña por defecto es admin*

Dentro de Gotify vamos a crear una App en mi caso la voy a llamar Alertas, esto nos dará un Token que es el que vamos a usar para conectar Gotify a Proxmox.

![1772013082633](image/Zimablade2/1772013082633.png)

Luego vamos a nuestro DataCenter Promox, vamos a configurar las notificaciones y a añadir nuestro Gotify.

![1772013182734](image/Zimablade2/1772013182734.png)

Aquí añades la URL y el token que acabamos de crear, con esto ya tienes Gotify vinculado y operativo para usar en alertas. Para probar que funciona puedes usar la función test que te ofrece Proxmox, este te enviara un mensaje de prueba si te llega es que funciona.

![1772013293463](image/Zimablade2/1772013293463.png)

En mi caso voy a crear diferentes targets y matchers para organizar las diferentes alertas pero es hacer lo mismo pero creando mas tokens.

![1772019122242](image/Zimablade2/1772019122242.png)

En Gotify se vería asi.

![1772019176934](image/Zimablade2/1772019176934.png)

### Uptime Kuma

Ahora para configurar alertas de nuestros servidores mas especificas como CPU, Disco, etc... vamos a implementar Uptime Kuma

Docker

```bash
docker run -d --restart=always -p 3001:3001 -v uptime-kuma:/app/data --name uptime-kuma louislam/uptime-kuma:2
```

Entramos a la interfaz web para empezar con la instalación `http://keepass:3001/`

Nos pide configurar la base de datos yo voy a elegir embedded maridadb, asi no tengo que configurar nada.

![1772017596506](image/Zimablade2/1772017596506.png)

Una vez termine de crearse la base de datos nos pedirá crear un usuario y una contraseña con todo esto ya podemos empezar a monitorizar todos los servidores o servicios que queramos

![1772018156893](image/Zimablade2/1772018156893.png)

Vamos a configurar Gotify para que envía las notificaciones, en mi caso lo voy a usar para monitorizar los servicios por lo que lo voy a conectar a mi chat dedicado a esto.

![1772019808154](image/Zimablade2/1772019808154.png)

Una vez configurado lo podemos testear, tendría que salir algo asi.

![1772019849017](image/Zimablade2/1772019849017.png)

Vamos a hacer una prueba con un servicio.

![1772020025832](image/Zimablade2/1772020025832.png)

Ahora lo voy a apagar el contenedor a ver si funciona y nos envía la notificación.

![1772020129263](image/Zimablade2/1772020129263.png)

Y cuando lo restablecemos también nos envía una notificación.

![1772020322338](image/Zimablade2/1772020322338.png)

Funciona, ahora la idea es hacer esto pero con todos los servicios que tenemos corriendo.

## N8N

### Docker

Vamos a crear el contenedor en docker.

```bash
docker volume create n8n_data
nano docker-compose.yml
"
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=automate.homelabeiro.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://automate.homelabeiro.com/
      - N8N_EDITOR_BASE_URL=https://automate.homelabeiro.com/
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
    external: true
"
```

Este servicio necesita HTTPS por lo que le vamos a configurar un reserve proxy

![1772615794072](image/Zimablade2/1772615794072.png)

Una vez creado entramos y ya podemos empezar a usar n8n

![1772615843823](image/Zimablade2/1772615843823.png)

### Meta Developers

La automatización que voy a preparar es envío de recordatorios por WhatsApp, para esto necesitamos la API de WhatsApp y por ello una cuenta en Meta Developers. Para esto voy a seguir la guía oficial de [Meta](https://developers.facebook.com/documentation/business-messaging/whatsapp/get-started).

Una vez creada y estando en el panel de mis aplicaciones vamos a crear nuestra primera aplicación.

![1772612058947](image/Zimablade2/1772612058947.png)

![1772612087213](image/Zimablade2/1772612087213.png)

Ahora nos manda crear un portfolio nuevo si no tenemos ninguno, vamos a crearlo.

![1772612224860](image/Zimablade2/1772612224860.png)

Una vez creado ya nos sale para seleccionarlo

![1772612333514](image/Zimablade2/1772612333514.png)

La configuración nos quedaría asi

![1772612372153](image/Zimablade2/1772612372153.png)

Ahora vamos a caso de uso y personalizamos el que seleccionamos al crear la API, y vamos a configurar la API.

![1772613107708](image/Zimablade2/1772613107708.png)

Ahora las credenciales que necesitamos para enviar mensajes son, el identificador de WhatsApp Busisness que se puede ver en la captura de arriba, y a mayares un usuario del sistema que tenga acceso total a esta App con el que crearemos el token con el que nos conectaremos.

![1772810151465](image/Zimablade2/1772810151465.png)

Una vez creado seria simplemente Generar el identificador y guardarnos el token de acceso para usarlo en N8N.

### Workflow

Vamos a configurar un workflow para que envié mensajes periódicos a los que se les pueda responder para saber si lo hiciste o no y asi llevar un seguimiento, y que si lo hiciste o no te envié un mensaje en respuesta a eso. El workflow básico que solo envía un mensaje todos los días a una hora exacta seria asi.

![1772809914047](image/Zimablade2/1772809914047.png)

#### Opción 1

Voy a integrar una IA local para clasificar respuestas por lo que voy a usar Ollama. Para no saturar este dispositivo voy a correr esta IA local en una Raspberry Pi 5 que tengo, lo puedes hacer instalando directamente ollama o con docker.

```bash
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Ahora nos vamos a meter en el contenedor y instalar llama3.2

```bash
docker exec -it ollama bash

ollama run llama3.2
```

Una vez en ollama en mi caso quiero personalizar la IA local para que responda como yo quiero que respondo para esto vamos a crear una IA personalizada a traves de un archivo Modelfile que le va a decir a la IA como tiene que responder.

```bash
FROM llama3.2

SYSTEM """
Eres un asistente cariñoso y gracioso. Respondes a si una chica se ha tomado su medicación diaria.

REGLAS:
- SOLO la frase, nada más.
- Máximo 5 palabras.
- Siempre en español.
- Máximo 1 emoji por respuesta.
"""

PARAMETER temperature 0.85
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.5
```

Ahora creamos la IA personalizada y probamos que funciona.

```bahs
ollama create pastilla-bot -f ./Modelfile
ollama run pastilla-bot "si me la tomé"
```

Ahora pasamos al flujo N8N que es el que va a recibir el mensaje y va a mandar la respuesta a la IA local y va a responder.

![1772891632271](image/Zimablade2/1772891632271.png)

#### Opción 2

Vamos a añadir un nodo Code para que genere una lista de respuestas ya que llama3.2 no es muy potente y tiene bastantes alucinaciones.

```bash
const frasesSi = [
  "Qué responsable 😌",
  "Eres la mejor 🥰",
  "Ya era hora 😏",
  "Menos mal 😅"
];

const frasesNo = [
  "Que no se te olvide eh 👀",
  "Dale, que es importante 🙄",
  "No me falles 😤",
  "Corre 💨💊"
];

const msg = $json.messages[0].text.body.toLowerCase();
const confirma = ["si","sí","sip","ya","tomada","listo","hecha","dale","amor"];
const esSi = confirma.some(p => msg.includes(p));

const lista = esSi ? frasesSi : frasesNo;
const frase = lista[Math.floor(Math.random() * lista.length)];

return [{ json: { ...($json), frase } }];
```

![1772895533394](image/Zimablade2/1772895533394.png)

Ahora para que yo pueda saber si se la tomo o no sin que ella me tenga que avisar voy a configurar un bot de telegram para que me envié por telegram el mensaje que ella le envié a mi bot de WhatsApp. *PD: no voy a explicar como se cree el bot de telegram ya que es algo que es muy sencillo de hacer y puedes buscar tu mismo en YouTube*

![1773926246775](image/Zimablade2/1773926246775.png)

El flujo quedaría asi ahora hay que configurar dentro del modulo de telegram el access token del Bot y añadir tu chat ID

![1773926322620](image/Zimablade2/1773926322620.png)

![1773926356680](image/Zimablade2/1773926356680.png)

Con todo esto ya podemos probar y ver que funciona el bot y envía el mensaje a telegram, debo aclarar que opte por esta opción ya que tengo problemas con el bot de WhatsApp y si no le respondes cada cierto tiempo te deja de enviar los mensajes, es como si entrara en modo suspensión.

###### Mejoras

Aquí voy a ir documentando todas las mejoras y actualizaciones de mi flujo, asi tengo un histórico de como lo fui mejorando.

![1774272801236](image/Zimablade2/1774272801236.png)

Mejore bastante el flujo y le añadí nuevas funcionalidades en resumen el flujo ahora te permite consultar cuantos días llevas tomando la medicación desde la última vez que te bajo todo a través de comandos de texto por WhatsApp, esto lo va almacenando en una base de datos con la que hace las consultas.

## UpSnap

Quiero ser capaz de poder apagar o encender los diferentes dispositivos de mi Homelab dese cualquier lado para esto vamos a configurar una Wake On LAN, aquí es donde entra este servicio UpSnap. Con este servicio vamos a poder configurar el apagado y encendido de nuestros servidores, ordenadores, etc...

### Docker

Vamos a levantar este servicio con Docker-Compose como usando el docker-compose de ejemplo.

```bash
services:
  upsnap:
    container_name: upsnap
    image: ghcr.io/seriousm4x/upsnap:5
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./data:/app/pb_data
    environment:                          # ← indentación corregida (estaba con 5 espacios extra)
      - TZ=Europe/Madrid
      - UPSNAP_HTTP_LISTEN=0.0.0.0:8090  # ← cambiado de 127.0.0.1 a 0.0.0.0
      - UPSNAP_INTERVAL=*/10 * * * * *
      - UPSNAP_SCAN_RANGE=192.168.1.0/24
      - UPSNAP_SCAN_TIMEOUT=500ms
      - UPSNAP_PING_PRIVILEGED=true
      - UPSNAP_WEBSITE_TITLE=HOMELAB
    entrypoint: /bin/sh -c "apk update && apk add --no-cache openssh-client && rm -rf /var/cache/apk/* && ./upsnap serve"
```

Ahora creamos el docker-compose.yml y lo levantamos

```bash
docker-compose up -d
```

Una vez levantado con `docker ps` podemos ver si se levanto bien o mal, si se levanto bien ya podemos acceder a la web.

```text
http://IP_DE_TU_SERVIDOR:8090
```

![1773232600332](image/Zimablade2/1773232600332.png)

### Configuración

Ahora seguimos los pasos para configurar la cuenta esto no tiene mucha complicación es crear una cuenta de inicio de sesión.

Vamos a añadir nuestro primer dispositivo, con este servicio tenemos dos opciones hacer un escaneo de nuestra red local en el rango que le digamos o rellenar los datos de forma manual, yo en mi caso voy a escanear mi red local y añadir los dispositivos que yo quiera.

![1773233284535](image/Zimablade2/1773233284535.png)

Una vez los añadimos ya los tenemos y los podemos editar.

![1773233372028](image/Zimablade2/1773233372028.png)

Es importante ahora aclarar que esto es el ultimo paso primero dentro de cada dispositivo que quieras añadir para usar Wake On LAN tienes que configurarlo previamente, normalmente esto se basa en activar la Wake On LAN en la BIOS del sistema de cada equipo, por lo que si solo haces esto no te funcionara.

En mi caso la Raspberry Pi no permite wake on lan por como funciona pero si le puedo configurar el apagado.

![1773235424380](image/Zimablade2/1773235424380.png)

# Ghost VM

Aquí vamos a crear una nueva VM el proceso es el mismo que en el resto, esta VM se va a usar para autohospedar nuestra web sobre nuestro Homelab y todos los tutoriales que publiquemos a traves de una newsletter. Usamos una VM ya que docker en las LXC tiene muchas limitaciones y funciona muy mal.

Descargamos la imagen ISO de Ubuntu24.04 desde la web [oficial]('https://releases.ubuntu.com/24.04/?_ga=2.149898549.2084151835.1707729318-1126754318.1683186906&_gl=1*fjkj0i*_gcl_au*MjA3Mzc4NjMzOC4xNzc3Mjg4ODkz'), yo en caso instale la version Server una vez instalada la subimos a proxmox en un storage.

![1777290236812](image/Zimablade2/1777290236812.png)

Con esto ya podemos iniciar la VM y empezar la instalación, en la instalacion se puede configurarar la IP estática tambien configuras el usuario y importante activar el servicio Openssh.

## Requisitos

* Docker y Docker Compose 20.10.13 o mayor
* Un Dominio con un DNS que apunte a este servidor
* Un servicio SMTP
* (Opcional) Cuenta en [TinyBird]('https://www.tinybird.co/')

Como primer paso vamos a iniciar sesión con el usuario y la contraseña que configuraste.

```bash
apt update && apt upgrade -y
```

Ahora ya podemos empezar con la instalación de Docker

## Docker

Para instalar docker vamos a seguir la Guía oficial de instalación que es la mejor manera y más segura.

[Instalación Docker]('https://docs.docker.com/engine/install/ubuntu/')

Una vez tenemos Docker y Docker compose ya podemos empezar la instalación de [Ghost]('https://docs.ghost.org/install/docker').

## Ghost

Empezamos instalando el repositorio.

```
span
```

Una vez clonado copiamos el archivo de ejemplo y lo configuramos para nuestro caso.

```
cp .env.example .env
cp caddy/Caddyfile.example caddy/Caddyfile
```

Yo en mi configuracion voy a comentar el SMTP temporalmente ya que no configure ninguno aun, ademas voy a usar Cloudfare que ya tengo configurado con mi dominio para que Ghost lo use por lo que no voy a hacer uso de Caddy.

Voy a configurar el compose.yml a mi caso en concreto por lo que todo lo que tengo que ver con caddy lo voy a comentar.

```
#  caddy:
#    image: caddy:2.10.2-alpine@sha256:953131cfea8e12bfe1c631a36308e9660e4389f0c3dfb3be957044d3ac92d446
#    restart: always
#    ports:
#      - "${HTTP_PORT:-80}:80"
#      - "${HTTPS_PORT:-443}:443"
#    environment:
#      DOMAIN: ${DOMAIN:?DOMAIN environment variable is required}
#      ADMIN_DOMAIN: ${ADMIN_DOMAIN:-}
#      ACTIVITYPUB_TARGET: ${ACTIVITYPUB_TARGET:-https://ap.ghost.org}
#    volumes:
#      - ./caddy:/etc/caddy
#      - caddy_data:/data
#      - caddy_config:/config
#    depends_on:
#      - ghost
#    networks:
#      - ghost_network



# Tambien los volumenes

volumes:
#  caddy_data:
#  caddy_config:
  tinybird_files:
  tinybird_home:
  traffic_analytics_data:
```

Tambien el ghost: vamos a añadir el puerto personalizado que le configuramos en el .env justo debajo de restart: always.

```
ports:
      - "2368:2368"
```

Una vez tenemos tanto el compose.yml, como el .env configurados a nuestro gusto podemos ejecutar el contenedor.

```
docker compose pull
docker compose up -d
```

Es útil mirar los logs y ver si esta haciendo las cosas bien o tiene errores.

```
sudo docker logs ghost-ghost-1 -f
```

Una vez iniciado puedes acceder a la URL y empezar la configuracion de tu sitio.

```
https://ghost.micloudm.duckdns.org/ghost/#/setup
```

Una vez vemos que con esta configuración básica funciona le vamos a configurar un servicio SMTP de prueba para poder iniciar sesion y enviar los primeros correos, en este caso voy a usar MailPit que lo vamos a lanzar junto al compose.yml.

```
mailpit:
    image: axllent/mailpit:latest
    restart: always
    ports:
      - "8025:8025"  # Interfaz web para ver los emails
    networks:
      - ghost_network
```

Luego configuramos las credenciales en el .env.

```
mail__transport=SMTP
mail__options__host=mailpit
mail__options__port=1025
mail__options__secure=false
mail__from="'Tu Blog' <noreply@tusubdominio.duckdns.org>"
```

ahora ya tenemos todas las funcionalidades basicas para crear nuestro blog pero nos hace falta que la conexión que hagan los visitantes sea segura por lo que vamos a crear un tunnel de cloudfare.

### Cloudfare

Ahora vamos a configurar un tunnel de Cloudfare para poder acceder a nuestro blog pero desde un dominio personalizado y seguro.

#### Requisitos

* Tener un dominio de tu propiedad o comprar uno
* Tener una cuenta de Cloudfare

Una vez tengas estos requisitos es fácil dese la web de cloudfare con nuestra cuenta iniciada vamos a Zero Trust.

Ahi dentro desde el apartado Networks creamos un tunnel lo nombramos como queramos y lo configuramos segun nuestro entorno, te va a pedir que instales cloudfared en mi caso lo instale en la VM con un docker run.

Una vez todo esto se veria algo asi

![1777303622932](image/Zimablade2/1777303622932.png)

Con esto ya nos podemos meter a nuestro dominio y ver que nos carga nuestro blog

![1777303658444](image/Zimablade2/1777303658444.png)

Con esto ya tenemos la web funcionando.

### Brevo

Ahora que ya tenemos todo funcional vamos a configurar el servidor SMTP definitivo, en mi caso voy a rpbar con Brevo ya que para empezar tiene un plan gratuito mas generoso y ahora mismo no tengo volumen para tener que usar Mailgun.

Lo primero que hay que hacer es crearse una cuenta gratuita en [Brevo]('https://brevo.com/').

Una vez tenemos la cuenta configurada simplemente tenemos que ir a la configuracion y en el apartado de SMTP y Api crear las claves para luego usarlas en nuestro .env

![1777366842498](image/Zimablade2/1777366842498.png)

Una vez la tienes creada es simplemente ir al .env y configurarlo de la siguiente manera.

```
mail__transport=SMTP
mail__options__host=smtp-relay.brevo.com
mail__options__port=587
mail__options__secure=false
mail__options__auth__user=tuemail@smtp-brevo.com
mail__options__auth__pass=TuClaveSmtpDeBrevo
mail__from="'Tu Blog' <tuemail@ejemplo.com>"
```

Con esto ya tenemos todo funcionando yo a mayores voy a añadir esta VM  a mi PBS para que haga Backups diarios de mi servidor por si pasa algo.

# Network-Services

Migre el servidor de mi PVE hacia mi PVE2 ya que con mi VM con ZimaOS mi PVE ya tiene mucha carga y mi PVE2 tiene menos carga, use la herramienta que viene integrada en Proxmox para migrar un contenedor de un cluster a otro.

## Cloudfared

Ahora cambiamos como estan expuestos mis servicios publicos, compre un dominio "homelabeiro.com" y a traves de tuneles de Cloudfare cree Routes y voy a cerrar los puertos de mi Router.

### Docker

Nos conectamos a traves de docker creando el servicio Cloudfared

```
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token
```

Luego simplemente creamos la ruta que apunta a todos nuestros nuevos subdominios que contengan nuestro dominio, y que esta apunte a nuestro NPM.

![1783068845664](image/Zimablade2/1783068845664.png)

Dentro de nuestro NPM creamos los nuevos Proxys con el nuevo dominio y el NPM redirige las conexiones a nuestros servicios.

Con este sencillo cambio ganamos en seguridad , comodidad y autocontrol.

## TTYD

Vamos a crear un contenedor docker con TTYD lo cual es una herramienta que te permite ejecutar terminales via WEB.

Un problema que hay con estas terminales es la persistencia de la informacion si actualizas la web pierdes la sesion actual por lo que para solucionar este problema voy a crear una imagen personalizada que tenga tmux integrado.

### DockerFile

```
FROM tsl0922/ttyd
RUN apt update && apt install -y openssh-client tmux sudo && \
    useradd -m -s /bin/bash mateo && \
    echo "mateo:TU_PASSWORD" | chpasswd && \
    usermod -aG sudo mateo

USER mateo
WORKDIR /home/mateo

CMD ["ttyd", "-W", "tmux", "new-session", "-A", "-s", "main", "bash"]
```

Por seguridad vamos a crear un usuario no root llamada wetty y a root le vamos a poner una contraseña, esto es opcional.

### Docker Compose

```
services:
  ttyd:
    build: .
    ports: ["7681:7681"]
    restart: unless-stopped
```

En mi caso voy a crear una terminal basica con bash tmux para persistencia de la sesion.

Como es una imagen modificada con un Dockerfile tener que hacer un build

`docker compose up -d --build`

Con esto corriendo ya podemos acceder a la terminal desde.

`http://IP-SERVER:7681/`

En mi caso la voy a integrar en mi Notion, para que funcionara tuve que crear un proxy con URL HTTPS, a esta url le configure una access list para que solo fuera accesible desde mi red local.

![1784552736436](image/Zimablade2/1784552736436.png)

## Whats Up Docker(WUD)

WUD es una herramienta para monitorizar tus servidores y mantenerlos siempre actualizados de una forma comoda. Esto es útil cuando tienes muchos contenedores.

### Docker Compose

```
services:
  whatsupdocker:
    image: getwud/wud
    container_name: wud
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - 3005:3000
```

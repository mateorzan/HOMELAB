
# KeePass Container LXC / DEPRECATED / MI

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

### Gotify

#### Docker

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

## Almacenamiento Media

Mi ZimaOS se estaba quedando sin espacio con tantas películas y series, y en la Raspberry tengo unos 800GB libres en el SSD externo. La idea es usar el disco de la Rasp como una segunda ubicación para las películas y series sin romper nada de lo que ya tengo montado: Radarr, Sonarr, Jellyfin y Seerr.

Hasta ahora el flujo era ZimaOS → Raspberry, es decir, ZimaOS comparte la carpeta de media por SMB y la Rasp la monta por CIFS para que Jellyfin la lea (explicado en [Servicios](./2-Servicios.md)). Ahora vamos a montar también el camino inverso, Raspberry → ZimaOS, para que Radarr y Sonarr puedan mover las pelis al disco de la Rasp.

```text
ZimaOS    ── SMB/CIFS ──▶  Raspberry: /home/mateorzan/media      (lo de siempre)
Raspberry ── Samba    ──▶  ZimaOS:    /DATA/media-pi             (lo nuevo)

Radarr   : /movies  (ZimaOS)  +  /movies-pi  (Raspberry)
Sonarr   : /tv      (ZimaOS)  +  /series-pi  (Raspberry)
Jellyfin : /media   (CIFS)    +  /media-pi   (disco local de la Raspberry)
Seerr    : no se toca nada
```

> IMPORTANTE: Seerr no mira los discos, solo pregunta a Jellyfin y a Radarr/Sonarr qué contenido hay. Por eso le da igual dónde esté el archivo. El que de verdad se rompe si movemos pelis a mano es Radarr, que las marca como ausentes y si están monitorizadas las vuelve a descargar. **Las pelis siempre se mueven desde Radarr y las series desde Sonarr, nunca a mano.**

### Requisitos

- Samba en la Raspberry
- cifs-utils en ZimaOS (ya viene)
- Acceso a la terminal de ZimaOS
- Radarr, Sonarr y Jellyfin funcionando

### CIFS vs Samba

Al principio me lió esto así que lo dejo apuntado. SMB/CIFS es el mismo protocolo para compartir carpetas en red, lo que cambia es el lado:

* **Samba** es el servidor, el programa que comparte la carpeta.
* **cifs** es el cliente, el tipo de montaje que usas para conectarte (`mount -t cifs`).

Hasta ahora ZimaOS hacía de servidor (trae Samba integrado) y la Rasp de cliente. Ahora la Rasp también tiene que compartir, por lo que necesita instalar Samba.

### Samba en la Raspberry

Creamos la carpeta que vamos a compartir con la estructura de películas y series.

```bash
mkdir -p "/home/mateorzan/media-pi/Movies" "/home/mateorzan/media-pi/TV Shows"
```

Instalamos Samba.

```bash
sudo apt update
sudo apt install samba smbclient
```

Editamos la configuración.

```bash
sudo nano /etc/samba/smb.conf
```

Añadimos al final del archivo.

```Shell
[media-pi]
   path = /home/mateorzan/media-pi
   browseable = yes
   read only = no
   valid users = mateorzan
   force user = mateorzan
   create mask = 0664
   directory mask = 0775
```

* **valid users**: solo nuestro usuario puede conectarse.
* **force user**: todo lo que llegue por Samba se guarda a nombre de `mateorzan` (uid 1000), que es el mismo usuario con el que corre Jellyfin, así no hay líos de permisos.
* **create mask 0664**: archivos de lectura y escritura para el dueño y el grupo, y solo lectura para el resto.
* **directory mask 0775**: igual pero para carpetas, que además necesitan permiso de ejecución para poder entrar en ellas.

En el mismo archivo comentamos con `;` las secciones `[homes]`, `[printers]` y `[print$]`. `[homes]` viene activado de serie y comparte todo el `/home` del usuario, cosa que no queremos.

```Shell
;[homes]
;   comment = Home Directories
;   browseable = no
;   ...
```

Creamos la contraseña de Samba para nuestro usuario. Es una contraseña aparte de la del sistema y es la que usaremos desde ZimaOS.

```bash
sudo smbpasswd -a mateorzan
```

Comprobamos que la configuración está bien, reiniciamos y lo dejamos activo al arranque.

```bash
testparm -s
sudo systemctl restart smbd
sudo systemctl enable smbd
```

Probamos desde la propia Rasp. Si salen `Movies` y `TV Shows` está todo listo.

```bash
smbclient //localhost/media-pi -U mateorzan -c 'ls'
```

### Montaje en ZimaOS

Desde la app Archivos de ZimaOS → **Cloud, LAN or insert USB** → LAN Storage, ponemos la IP de la Rasp, quitamos **Connect as guest** y metemos el usuario y la contraseña de Samba.

Esto funciona y se ve en la app de Archivos, pero aquí me encontré con un problema. Si miramos cómo lo ha montado:

```bash
mount | grep cifs
```

```text
//192.168.1.62/media-pi on /media/192.168.1.62/media-pi type cifs (rw,...,uid=0,gid=0,...,file_mode=0755,dir_mode=0755,...)
```

ZimaOS lo monta como **root** (`uid=0,gid=0`) con permisos `0755`, es decir, solo root puede escribir. Radarr y Sonarr corren con `PUID=1000` así que no pueden mover nada ahí.

> El usuario y contraseña de SMB solo sirven para autenticarse en la Rasp. El dueño que ves en ZimaOS lo deciden las opciones del montaje. Tampoco sirve crear un enlace, un `mount --bind` o hacer `chown` ya que heredan los permisos del montaje CIFS.

La solución es montar el mismo recurso otra vez en otra ruta pero con nuestro usuario (uid 1000). CIFS permite montar lo mismo varias veces sin problema.

```bash
sudo mkdir -p /DATA/media-pi
sudo mount -t cifs //IP_RASPBERRY/media-pi /DATA/media-pi -o username=tu_usuario,uid=1000,gid=1000,file_mode=0664,dir_mode=0775,vers=3.1.1,iocharset=utf8
```

Ejemplo

`sudo mount -t cifs //192.168.1.62/media-pi /DATA/media-pi -o username=****,uid=1000,gid=1000,file_mode=0664,dir_mode=0775,vers=3.1.1,iocharset=utf8`

Comprobamos que ahora las carpetas son de nuestro usuario.

```bash
ls -la /DATA/media-pi
```

Así nos quedan dos montajes:

* `/media/192.168.1.62/media-pi` → como root, solo para verlo desde la app de Archivos.
* `/DATA/media-pi` → como 1000, el que usan Radarr y Sonarr.

#### Montaje automático

Igual que hicimos en la Rasp, creamos un archivo para las credenciales. Lo dejo en `/DATA` que es la parte de ZimaOS que se mantiene.

`sudo nano /DATA/.smb-mediapi`

```Shell
username=tu_usuario
password=tu_contraseña_samba
```

Lo protegemos.

`sudo chmod 600 /DATA/.smb-mediapi`

Y lo añadimos al fstab.

`sudo nano /etc/fstab`

`//IP_RASPBERRY/media-pi  /DATA/media-pi  cifs  credentials=/DATA/.smb-mediapi,uid=1000,gid=1000,file_mode=0664,dir_mode=0775,vers=3.1.1,iocharset=utf8,_netdev,nofail  0  0`

Probamos que no da ningún error.

`sudo mount -a`

> PENDIENTE: ZimaOS tiene parte del sistema de solo lectura y no tengo claro que `/etc/fstab` sobreviva a un reinicio o una actualización. Hay que reiniciar y comprobarlo, si no aguanta lo montamos con un servicio de arranque.

> Si la Rasp está apagada cuando arranca ZimaOS, Radarr verá la carpeta vacía. Cuando vuelva la Rasp hacemos `sudo mount -a` y reiniciamos Radarr y Sonarr.

### Radarr y Sonarr

Añadimos el nuevo volumen en el compose de cada uno (en mi caso desde la interfaz de ZimaOS, en la sección Mount).

| App | Host | Container |
| :----: | :-----------------------: | :----------: |
| Radarr | `/DATA/media-pi/Movies` | `/movies-pi` |
| Sonarr | `/DATA/media-pi/TV Shows` | `/series-pi` |

> No uses la ruta que te enseña la app de Archivos (`/192.168.1.62/media-pi/...`), esa ruta no existe en el sistema y Docker monta una carpeta vacía. La ruta real se saca con `mount | grep cifs`.

Comprobamos que el contenedor puede escribir con su usuario.

```bash
docker exec -u 1000 radarr touch /movies-pi/prueba && docker exec radarr rm /movies-pi/prueba
```

Si no da error ya podemos añadir la carpeta en Radarr.

Radarr → Settings → Media Management → Root Folders

  Click **Add Root Folder**

   Rellena:

     **Path**: `/movies-pi`

**OK** — debería aparecer con el espacio libre del disco de la Rasp.

> La lista de Root Folders no se rellena sola al añadir el volumen, hay que añadirla a mano.

En la misma página comprobamos que **Unmonitor Deleted Movies** está desmarcado.

En Sonarr lo mismo pero con `/series-pi`.

#### Mover películas

Radarr → Movies → seleccionamos la película → **Edit**

   **Root Folder**: `/movies-pi`

**Save** → cuando pregunte, **"Sí, mover los archivos"**

Radarr copia el archivo por la red y borra el original. Para mover varias de golpe usamos el modo selección / editor masivo y hacemos lo mismo.

Con las series igual desde Sonarr.

> Las descargas nuevas siguen yendo a la carpeta raíz por defecto (ZimaOS). Si queremos que vayan a la Rasp lo elegimos al añadir la peli o lo cambiamos en Seerr → Settings → Services. Ojo que entre Deluge y la Rasp no hay hardlinks, el archivo se copia y mientras siga en seeding ocupa el doble.

### Jellyfin

En la Rasp el disco nuevo es local así que no hace falta SMB, solo añadirlo al compose. De paso limpié el compose, tenía `Movies` y `'TV Shows'` montados por separado cuando ya montaba toda la carpeta `media` entera (y las comillas simples en YAML se quedan como parte del nombre de la carpeta).

```Shell
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    # Optional - specify the uid and gid you would like Jellyfin to use instead of root
    user: 1000:1000
    ports:
      - 8096:8096
      - 7359:7359
    volumes:
      - /srv/jellyfin/config:/config
      - /srv/jellyfin/cache:/cache
      - type: bind
        source: /home/mateorzan/media
        target: /media
      - type: bind
        source: /home/mateorzan/media-pi
        target: /media-pi
    restart: 'unless-stopped'
    # Optional - alternative address used for autodiscovery
    environment:
      - JELLYFIN_PublishedServerUrl=https://RaspiTV.example.com/
    # Optional - may be necessary for docker healthcheck to pass if running in host network mode
    extra_hosts:
      - 'host.docker.internal:host-gateway'
```

```bash
docker compose up -d
```

En Jellyfin añadimos las rutas nuevas a las bibliotecas que ya tenemos, **no creamos bibliotecas nuevas** así Seerr sigue viendo lo mismo sin tocar nada.

* Películas → añadir carpeta `/media-pi/Movies`
* Series → añadir carpeta `/media-pi/TV Shows`

### Comprobación

Después de mover la primera peli:

```bash
ls "/home/mateorzan/media-pi/Movies"
```

* En la Rasp aparece la carpeta de la peli.
* En Jellyfin escaneamos la biblioteca y la peli sigue apareciendo y se reproduce.
* En Seerr sigue apareciendo como disponible.

Si todo va bien ya podemos ir moviendo por lotes.

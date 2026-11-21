## Seguridad

Raspberry Pi OS viene prácticamente sin protección, no trae firewall configurado, ni fail2ban, ni actualizaciones automáticas. En mis servidores Ubuntu ya tengo UFW, fail2ban y unattended-upgrades así que vamos a montar lo mismo en la Rasp, con algún cambio ya que es Debian 13 (trixie).

### Revisión inicial

Antes de tocar nada vamos a ver cómo está la máquina.

```bash
cat /etc/os-release | grep PRETTY        # versión
sudo nft list ruleset                    # reglas de firewall
which ufw fail2ban-client                # ¿instalados?
sudo ss -tulpn                           # qué puertos escuchan
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin|pubkeyauthentication"
```

En mi caso:

* La cadena `INPUT` estaba en `policy accept`, solo con las reglas que mete Docker y Tailscale, es decir, sin ningún filtro.
* Ni UFW ni fail2ban instalados.
* SSH con `passwordauthentication yes`.

Y estos eran los puertos abiertos.

|   Puerto   |        Servicio        |                    Qué hacemos                    |
| :---------: | :--------------------: | :------------------------------------------------: |
|     22     |          SSH          |                Solo LAN y Tailscale                |
|  445 / 139  |          Samba          |                  Solo desde ZimaOS                  |
|  137 / 138  |      nmbd (NetBIOS)      |                    Desactivado                    |
|    5403    |     corosync-qnetd     | Solo pve y pve2 ([QDevice](./4-QDevice.md)) |
| 2049, 111... |    NFS + rpcbind    | Solo pve y pve2 ([Backups](./3-Backups.md)) |
|    5900    |      wayvnc (VNC)      |                  Solo Tailscale                  |
|     631     |          CUPS          |                    Desactivado                    |
| 8096, 7359, 3000, 3100 |      Docker      |   Accesibles en la LAN (Docker se salta UFW)   |
| 11434, 18789 | Ollama, OpenClaw |           Solo localhost, no expuestos           |

### Desactivar servicios

Apagamos lo que no usamos.

```bash
sudo systemctl disable --now nmbd                 # NetBIOS, no hace falta
sudo systemctl disable --now cups cups-browsed    # impresión
```

### NFS

El NFS de los [Backups](./3-Backups.md) estaba compartido con toda la red local y con `no_root_squash`, lo que quiere decir que cualquier equipo de mi casa podía escribir como root en esa carpeta. Lo limitamos solo a los dos nodos de Proxmox.

`sudo nano /etc/exports`

```Shell
/ruta/backups 192.168.1.47(rw,sync,no_subtree_check,no_root_squash) 192.168.1.48(rw,sync,no_subtree_check,no_root_squash)
```

```bash
sudo exportfs -ra
sudo exportfs -v
```

> `no_root_squash` se mantiene porque Proxmox escribe las backups como root, sin él da errores de permisos.

Antes de tocar nada comprobé que había clientes conectados, así vi que pve y pve2 lo estaban usando.

```bash
sudo ss -tn state established '( sport = :2049 )'
```

### UFW

Instalamos UFW.

```bash
sudo apt update
sudo apt install ufw
```

> IMPORTANTE: Antes de activar UFW deja otra sesión SSH abierta por si algo falla.

Por defecto denegamos todo lo que entra y permitimos todo lo que sale, y luego abrimos solo lo necesario.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow in on tailscale0                                   # todo lo que llega por Tailscale
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp       # SSH desde la LAN
sudo ufw allow from 192.168.1.38 to any port 445 proto tcp        # Samba solo desde ZimaOS
sudo ufw allow from 192.168.1.47 to any port 2049 proto tcp       # NFS backups pve
sudo ufw allow from 192.168.1.48 to any port 2049 proto tcp       # NFS backups pve2
sudo ufw allow from 192.168.1.47 to any port 5403 proto tcp       # QDevice pve
sudo ufw allow from 192.168.1.48 to any port 5403 proto tcp       # QDevice pve2
sudo ufw allow from 192.168.1.0/24 to any port 5353 proto udp     # mDNS

sudo ufw --force enable
sudo ufw status verbose
```

Resultado.

```text
Status: active
Default: deny (incoming), allow (outgoing), deny (routed)

To                          Action      From
--                          ------      ----
Anywhere on tailscale0      ALLOW IN    Anywhere
22/tcp                      ALLOW IN    192.168.1.0/24
445/tcp                     ALLOW IN    192.168.1.38
2049/tcp                    ALLOW IN    192.168.1.47
2049/tcp                    ALLOW IN    192.168.1.48
5403/tcp                    ALLOW IN    192.168.1.47
5403/tcp                    ALLOW IN    192.168.1.48
5353/udp                    ALLOW IN    192.168.1.0/24
Anywhere (v6) on tailscale0 ALLOW IN    Anywhere (v6)
```

* **VNC**: el puerto 5900 solo queda accesible por Tailscale. Si lo necesitas desde la LAN: `sudo ufw allow from 192.168.1.0/24 to any port 5900 proto tcp`.
* **Docker**: los puertos que publica un contenedor (`8096:8096`) **se saltan UFW** porque Docker mete sus propias reglas en iptables. Si queremos restringirlos hay que publicarlos en una IP concreta (`192.168.1.62:8096:8096`) o usar `ufw-docker`. Con los puertos del router cerrados no es grave pero hay que saberlo.

### Fail2ban

Instalamos fail2ban.

```bash
sudo apt install fail2ban python3-systemd
```

Creamos la configuración.

`sudo nano /etc/fail2ban/jail.local`

```Shell
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 192.168.1.0/24 100.64.0.0/10

[sshd]
enabled = true
backend = systemd
```

* **bantime**: tiempo de baneo.
* **findtime / maxretry**: 5 fallos en 10 minutos y baneado.
* **ignoreip**: localhost, mi LAN y la red de Tailscale, así no me baneo a mí mismo.

> La diferencia con Ubuntu es `backend = systemd`. En Debian 13 no existe `/var/log/auth.log`, todo va al journal y sin esta línea fail2ban no arranca.

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

### Actualizaciones automáticas

Instalamos unattended-upgrades.

```bash
sudo apt install unattended-upgrades apt-listchanges
```

Activamos las actualizaciones diarias.

`sudo nano /etc/apt/apt.conf.d/20auto-upgrades`

```Shell
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Por defecto solo instala los parches de seguridad de Debian, si queremos que actualice también los paquetes propios de Raspberry Pi (kernel, firmware) creamos este archivo.

`sudo nano /etc/apt/apt.conf.d/51unattended-upgrades-rpi`

```Shell
Unattended-Upgrade::Origins-Pattern:: "origin=Raspberry Pi Foundation,codename=${distro_codename},label=Raspberry Pi Foundation";
```

Comprobamos que funciona (tarda un rato, es solo una simulación).

```bash
sudo unattended-upgrade --dry-run --debug
```

### SSH

Por ahora mantengo el acceso con contraseña, queda limitado a la LAN y Tailscale y con fail2ban activo así que el riesgo es bajo.

`PermitRootLogin` lo dejo en `without-password` (root solo con clave) ya que Proxmox lo usa si hay que volver a hacer `pvecm qdevice setup`, explicado en [QDevice](./4-QDevice.md).

Cuando quiera pasar a solo clave:

Desde mi PC copiamos la clave.

> `ssh-copy-id mateorzan@IP_RASPBERRY`

Comprobamos que entramos con clave y creamos el archivo.

`sudo nano /etc/ssh/sshd_config.d/hardening.conf`

```Shell
PasswordAuthentication no
KbdInteractiveAuthentication no
```

```bash
sudo sshd -t && sudo systemctl restart ssh
```

> IMPORTANTE: Antes de cerrar la sesión abre otra nueva para comprobar que entras.

### Comprobación

```bash
sudo ss -tulpn | grep LISTEN
sudo ufw status verbose
sudo fail2ban-client status sshd
```

Y desde pve comprobamos que el QDevice sigue votando y lanzamos una backup manual hacia la Rasp.

```bash
pvecm status
```

### Pendiente

* Ver la versión de NFS con `mount | grep nfs` en pve. Si es v4 desactivar `rpcbind` (`sudo systemctl disable --now rpcbind rpcbind.socket && sudo systemctl mask rpcbind rpcbind.socket`), si es v3 forzar antes `options vers=4.2` en el storage de Proxmox.
* Jellyfin: cambiar `7359:7359` por `7359:7359/udp`, el autodescubrimiento va por UDP.
* Publicar los puertos de Docker solo en la IP de la LAN.
* Pasar SSH a solo clave.

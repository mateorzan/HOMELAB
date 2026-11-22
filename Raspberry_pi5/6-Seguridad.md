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

En una instalación recién hecha lo normal es que la cadena `INPUT` esté en `policy accept` (solo con las reglas que meten Docker y Tailscale) y que no haya ni UFW ni fail2ban, es decir, sin ningún filtro. Con `ss -tulpn` vemos qué servicios escuchan y cuáles sobran. Los servicios que necesitan acceso desde otros equipos (SSH, Samba, NFS para los [Backups](./3-Backups.md) con los puertos 2049 y 111, el [QDevice](./4-QDevice.md)) los dejamos solo para las IPs que los usan y el resto se bloquea.

### Desactivar servicios

Apagamos lo que no usamos (NetBIOS, impresión...).

```bash
sudo systemctl disable --now nmbd                 # NetBIOS, no hace falta
sudo systemctl disable --now cups cups-browsed    # impresión
```

### NFS

El NFS de los [Backups](./3-Backups.md) no debería estar compartido con toda la red local, y menos con `no_root_squash`, porque cualquier equipo podría escribir como root en esa carpeta. Lo limitamos solo a los dos nodos de Proxmox.

`sudo nano /etc/exports`

```Shell
/ruta/backups IP_PVE(rw,sync,no_subtree_check,no_root_squash) IP_PVE2(rw,sync,no_subtree_check,no_root_squash)
```

```bash
sudo exportfs -ra
sudo exportfs -v
```

> `no_root_squash` se mantiene porque Proxmox escribe las backups como root, sin él da errores de permisos.

Antes de tocar nada conviene comprobar qué clientes están conectados para no cortar a nadie.

```bash
sudo ss -tn state established '( sport = :2049 )'
```

#### NFSv4 en Proxmox

Con NFSv3 el servidor usa puertos aleatorios para `mountd`, y habría que fijarlos y abrirlos en UFW. Para no tener que hacerlo forcé NFSv4 en el storage, que solo necesita el 2049 (más el 111 de rpcbind, explicado en [UFW](#ufw)). Estos comandos se ejecutan en pve.

```bash
pvesm set Rasp_bks --options vers=4.2
rpcinfo -T tcp IP_RASPBERRY nfs 4
pvesm status | grep Rasp_bks
```

El storage debe salir como `active`.

> `rpcbind` se queda activo en la Raspberry aunque usemos NFSv4, porque Proxmox lo sigue necesitando para comprobar el storage.

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
sudo ufw allow from IP_ZIMAOS to any port 445 proto tcp        # Samba solo desde ZimaOS
sudo ufw allow from IP_PVE to any port 2049 proto tcp       # NFS backups pve
sudo ufw allow from IP_PVE2 to any port 2049 proto tcp       # NFS backups pve2
sudo ufw allow from IP_PVE to any port 111                   # rpcbind pve (Proxmox comprueba el NFS)
sudo ufw allow from IP_PVE2 to any port 111                  # rpcbind pve2
sudo ufw allow from IP_PVE to any port 5403 proto tcp       # QDevice pve
sudo ufw allow from IP_PVE2 to any port 5403 proto tcp       # QDevice pve2
sudo ufw allow from 192.168.1.0/24 to any port 5353 proto udp     # mDNS

sudo ufw --force enable
sudo ufw status verbose
```

> IMPORTANTE: Aunque el 2049 esté abierto, hay que abrir también el **111** (rpcbind). Proxmox comprueba cada ~20 segundos si el servidor NFS está disponible preguntando a rpcbind, y si UFW bloquea ese puerto marca el storage como inactivo (`Active: No`) y las backups fallan con `could not activate storage`. Yo lo detecté mirando el log del firewall, donde salían bloqueos con `DPT=111` desde pve:
>
> `sudo journalctl -k --since "today" | grep "UFW BLOCK" | grep -E "SRC=IP_PVE|SRC=IP_PVE2"`

* **Servicios extra**: cualquier otro servicio (por ejemplo VNC) queda bloqueado hasta que lo abras con su propia regla.
* **Docker**: los puertos que publica un contenedor (`8096:8096`) **se saltan UFW** porque Docker mete sus propias reglas en iptables. Si queremos restringirlos hay que publicarlos en una IP concreta (`IP_RASPBERRY:8096:8096`) o usar `ufw-docker`.

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

Lo recomendable es acceder solo con clave, limitar SSH a la LAN y Tailscale (ya hecho con UFW) y tener fail2ban activo.

`PermitRootLogin` se deja en `without-password` (root solo con clave), ya que Proxmox lo usa si hay que volver a hacer `pvecm qdevice setup`, explicado en [QDevice](./4-QDevice.md).

Para pasar a solo clave, desde nuestro PC copiamos la clave.

> `ssh-copy-id tu_usuario@IP_RASPBERRY`

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

### Mejoras posibles

* Publicar los puertos de Docker solo en la IP de la LAN.
* Pasar SSH a solo clave.

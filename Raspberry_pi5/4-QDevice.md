# QDevice para Proxmox Quorum

Voy a configurar mi raspberry como QDevice, ¿Que es un QDevice?, es un servicio ligero y externo que actúa como un **tercer voto** o árbitro en un clúster de alta disponibilidad, esto es realmente útil en nodos como el mio de 2 equipos ya que ayuda como desempate y ademas si uno se cae el nodo cluster no pierde el Quorum.

## Requisitos previos

* Clúster Proxmox ya formado con 2 nodos (en este caso pve y pve2)
* Raspberry Pi 5 con Raspberry Pi OS de 64 bits (o Debian/Ubuntu ARM64) — la arquitectura de 32 bits no sirve para corosync-qnetd
* Acceso root por SSH a la Pi y a ambos nodos Proxmox
* La Pi debe tener IP fija y ser alcanzable desde ambos nodos (misma LAN o vía Tailscale)

## Configuración

### Paso 1 — Prepara la Raspberry Pi 5

Instala Raspberry Pi OS Lite (64-bit) o Debian/Ubuntu ARM64. Actualiza el sistema:

```
apt update && apt upgrade -y
```

Confirma que tienes una IP fija asignada (por DHCP reservation o configuración estática) y que la Pi responde a ping desde pve y pve2.

### Paso 2 — Instala `corosync-qnetd` en la Pi

```bash
sudo apt install corosync-qnetd -y
```

Verifica que el servicio está activo:

```bash
systemctl status corosync-qnetd
```

> Si el paquete no está disponible en los repos por defecto, instala desde los repos de Debian/Raspbian estándar — es totalmente compatible con Proxmox.

### Paso 3 — Configura acceso SSH sin contraseña

pvecm qdevice setup se conecta siempre como root@<IP-Pi></ip> — no admite otro usuario. Si el login de root por SSH está deshabilitado en la Pi, la forma más simple es habilitarlo temporalmente con contraseña, copiar tu clave con ssh-copy-id, y luego restringirlo a solo-clave (o desactivarlo del todo).

#### 3.1 — En la Raspberry Pi, habilita root con contraseña temporalmente

```Shell
sudo nano /etc/ssh/sshd_config
```

Busca (o añade) la línea:

```Shell
PermitRootLogin yes
```

Asegúrate también de que root tiene una contraseña puesta (si nunca la configuraste):

```Shell
sudo passwd root
```

Reinicia el servicio SSH:

```Shell
sudo systemctl restart sshd
```

#### 3.2 — Desde pve, genera (si no tienes ya) tu clave SSH y cópiala a root en la Pi

```Shell
ssh-keygen -t ed25519   # si no tienes clave ya
ssh-copy-id root@
```

Te pedirá la contraseña de root que acabas de establecer — con eso instala tu clave pública en /root/.ssh/authorized_keys automáticamente.
<IP-Pi></ip>

#### 3.3 — Vuelve a restringir el acceso de root a solo-clave editando otra vez /etc/ssh/sshd_config en la Pi

```Shell
PermitRootLogin prohibit-password

sudo systemctl restart sshd
```

#### 3.4 — Verifica desde pve que entras como root sin que te pida contraseña

```Shell
ssh root@<IP-RASP>
```

Si entra directamente, está listo para el Paso 5.

Una vez completado el setup del QDevice (Paso 5), puedes volver a poner `PermitRootLogin no` en la Pi y reiniciar sshd — el QDevice ya no necesita acceso SSH tras la configuración inicial, solo el servicio corosync-qnetd corriendo.
<IP-Pi></ip>

## Paso 4 — Instala `corosync-qdevice` en ambos nodos Proxmox

En `pve` **y** en `pve2`:

```bash
apt install corosync-qdevice -y
```

Antes de continuar, comprueba que el clúster está sano:

```bash
pvecm status
```

## Paso 5 — Añade el QDevice al clúster

Desde cualquier nodo (por ejemplo `pve`):

```
pvecm qdevice setup  -f
```

Este comando se conecta por SSH a la Pi, instala la configuración de QNetd, genera los certificados necesarios y reinicia corosync en los nodos del clúster. El flag -f fuerza el setup si ya hubiera una configuración QDevice previa.

## Paso 6 — Verifica el quórum

En `pve` o `pve2`:

```Shell
pvecm status
```

Deberías ver 3 votos totales (1 por cada nodo Proxmox + 1 del QDevice) y Quorate: Yes.

Para comprobar el estado de conexión específico del QDevice:

```
corosync-qdevice-tool -s
```

## Paso 7 — Prueba de failover

Apaga uno de los dos nodos Proxmox (pve o pve2) y verifica que el nodo que queda en pie mantiene quórum (2 de 3 votos) en pvecm status, en lugar de bloquearse como ocurriría sin QDevice en un clúster de solo 2 nodos.
<IP-Pi></ip>

## Notas y cosas a vigilar

* El QDevice no depende del tipo de storage de los nodos (ZFS, local, etc.) — solo necesita conectividad de red estable y de baja latencia hacia la Pi.
* Si usas Tailscale entre tus máquinas, esa misma red sirve perfectamente para la comunicación con el QDevice.
* Si `apt install corosync-qnetd` falla desde el repo de Proxmox en ARM, usa el paquete equivalente de los repos de Debian/Raspbian — es compatible.
* La Pi no forma parte del clúster de Proxmox ni almacena datos de él: solo aporta un voto de quórum vía `corosync-qnetd`.
* No olvides revertir `PermitRootLogin no` en la Pi (y reiniciar `sshd`) una vez completado el Paso 5, si solo lo habilitaste para el setup inicial del QDevice.

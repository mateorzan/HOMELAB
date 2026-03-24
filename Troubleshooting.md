# TroubleShooting

Como la raspberry y Zima tienen diferentes arquitecturas no todos los servicios se pueden ejecutar en los dos servidores, por lo que hay servicios que no son multi-arch que hay que ejecutar por separado.

Tuve problemas para montar el servicio nfs como un servicio en docker swarm fíjate que todas las rutas, la els y imágenes estén bn elegidas y sean compatibles.

Debido a los problemas y poco soporte viste en docker swarm vamos a empezar de 0 y probar a instalar K3s, el cual es una distribución mas ligera de Kubernetes pensada para ahorrar memoria y recursos del sistema, lo cual es justo lo que necesitamos.

## CasaOS

### Homepage No funciona URL Web

Reinstalar CasaOS

``` bash
curl -fsSL https://get.casaos.io/uninstall | bash
```

Borrar CasaOS

``` bash
curl -fsSL https://get.casaos.io | bash
```

Instalar CasaOS

``` bash
reboot
```

Reiniciar.

*Si esto sigue sin funcionar y crees que es un problema de puertos.*

``` bash
cd /etc/casaos
```

``` bash
nano gateway.ini
```

*Entramos a la configuración del puerto y lo cambiamos por uno disponible, ej: port=90.*

``` bash
sudo systemctl restart casaos-gateway
```

### Nextcloud borra carpeta DATA al instalarlo en CasaOS

Hay que primero crear la aplicación en CasaOS y luego hacer la sincronización de la carpeta DATA.

``` bash
sudo systemctl stop casaos
sudo systemctl stop docker
```

Paramos los servicios.

``` bash
sudo rsync -avh --progress /mnt/rpi/DATA/AppData/big-bear-nextcloud/ /DATA/AppData/big-bear-nextcloud/
```

Copiamos la carpeta DATA pero en este caso solo la de Nextcloud, que el resto no tuvimos problemas de borrado.

``` bash
sudo systemctl start casaos
sudo systemctl start docker
```

Una vez copiado iniciamos todo y empezara a migrar los datos.

### Nginx Proxy Manager

Para que este servicio funcione como lo tenemos configurado en mi HOMELAB primero hay que abrir los puertos 80 y 443 del router para la ip de este servidor.

``` bash
http://192.168.1.1/
```

La ruta para acceder a tu router suele ser esta.

Importante también hay que cambiar el puerto del CasaOS ya que por defecto usa el 80, en mi caso le configure el 90 para la pagina de inicio.

![1766580833807](image/README/1766580833807.png)

Una vez configurado los pueros del router hay que modificar los Proxy Hosts ya que están configurados para la ip del servidor anterior y hay que configurable la IP de este nuevo servidor para que funcionen.

![1766580747899](image/README/1766580747899.png)

Para que el proxy de Nextcloud también funcione hay que editar el siguiente archivo de configuración con la ip de este servidor.

``` bash
/DATA/AppData/big-bear-nextcloud/html/config/config.php
```

![1766581128725](image/README/1766581128725.png)

### Nextcloud Problema BD Postgres

La aplicación Nextcloud no era capaz de iniciarse  ya que daba un error de que la base de datos estaba unhealthy.

Para solucionar este error probamos a borrar la carpeta /pgdata de nextcloud, esta carpeta solo contiene la información de la base de datos por lo que no perdemos información ni datos como tal, todos los datos o archivos están almacenados en /html/data.

Una vez borrado instalamos nextcloud otra vez y ahora no nos dio error, pero no consigue iniciar, esto se debe a que le falta los datos de las tablas para poder inicia, para ello tuvimos que copiar las tablas y los datos de la base de datos de la Raspberry.

``` bash
docker exec -i db-postgres pg_dump -U nextcloud nextcloud > nextcloud.sql

scp nextcloud.sql user@zima:/ruta/destino/
```

``` bash
sed -i 's/oc_admin/casaos/g' /ruta/destino/nextcloud.sql

docker exec -i db-nextcloud psql -U casaos nextcloud < /ruta/destino/nextcloud.sql
```

En el archivo config.php tuvimos que cambiar el usuario y la contraseña con el que se conecta a la db por el usuario y contraseña que tenemos configurado en la pantalla de instalación de la db postgres, ademas tuvimos que activar la actualización via web, modificando el archivo upgrade-disable.

### Nextcloud Problema iniciar sesión en cliente desktop

No conseguía iniciar sesión se quedaba pillado, para solucionar esto tuve que hacer que el servicio solo fuera accesible completamente local.

![1766678767628](image/README/1766678767628.png)

Primero desactive el proxy para Nextcloud.

![1768053813964](image/README/1768053813964.png)

![1768053837389](image/README/1768053837389.png)

Luego en el archivo config.php de la carpeta html/config hay que editar toda esta configuración de esta manera.

![1768053862958](image/README/1768053862958.png)

![1768053894262](image/README/1768053894262.png)

Con esto ya debería de funcionar luego ya puedes restablecer la configuración como estaba antes y asi se pueda acceder online.

![1766678981799](image/README/1766678981799.png)

### Sonarr Problema permisos

No importaba los episodios por que le faltaba permisos en la carpeta /tv, para ello ejecutamos los siguientes comandos.

``` bash
sudo chown -R 1000:1000 /DATA/Media/TV
sudo chown -R 1000:1000 /DATA/Downloads
sudo chmod -R 775 /DATA/Media/TV
sudo chmod -R 775 /DATA/Downloads
```

*Esto tiene que estar adaptado a tus rutas concretas.*

### Almacenamiento, incremento de disco de la VM

Queremos aumentar el almacenamiento para esto tuvimos que usar estos comandos dentro de la terminal de la VM.

``` bash
sudo systemctl stop docker
sudo systemctl stop casaos
```

Paramos tanto docker como casaos para asi evitar posibles problemas.

``` bash
lsblk
sudo growpart /dev/sda 3
lsblk
```

Expandimos la partición.

``` bash
sudo pvresize /dev/sda3
sudo vgs
```

Hacemos que la VM vea el nuevo almacenamiento.

``` bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Expandimos y redimensionamos el almacenamiento.

``` bash
sudo systemctl start docker
sudo systemctl start casaos
```

Iniciamos todo.

### Error al cambiar a IP fija, no ves la red lan

Al cambiar la red de mi PC a una ip fija local no conseguí acceder a mi CasaOS para esto tuve que hacer lo siguiente.

Abrir el CMD como administrador y ejecutar los siguientes comandos

``` bash
route print
```

buscamos una linea como esta

``` bash
192.168.1.0    255.255.255.0      En vínculo      192.168.1.50   281
```

Si en puerta de enlace nos sale En vínculo tenemos que modificarla para q apunte a nuestro router con los siguientes comandos.

``` bash
route delete 192.168.1.0 mask 255.255.255.0
route add 192.168.1.0 mask 255.255.255.0 192.168.1.1 metric 1 -p
```

Una vez echo esto hacemos ping a nuestro servidor y vemos que ya tenemos conexión

``` bash
ping IP_CASAOS
```

## Tailscale

### Error no me deja instalar Tailscale en Proxmox, hay que editar el siguiente archivo para que instale los programas desde un repositorio gratuito

``` bash
nano /etc/apt/sources.list.d/pve-enterprise.list
```

Comente la linea que aparece en y añade el repositorio que no tiene subscription, luego actualiza los paquetes.

``` bash
# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise

echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update
```

Hay que hacer lo mismo con el siguiente archivo.

``` bash
nano /etc/apt/sources.list.d/ceph.list

# deb https://enterprise.proxmox.com/debian/ceph-quincy bookworm InRelease

echo "deb http://download.proxmox.com/debian/ceph-quincy bookworm main" > /etc/apt/sources.list.d/ceph-no-subscription.list
apt update
```

## VirtualBox

### Activar virtualización en BIOS

VIRTUALBOX nos da el siguiente error al intentar iniciar la VM.

VT-x is disabled in the BIOS

Para que la VM funcione necesitamos activar esta opción en la BIOS, para esto hay que reiniciar el dispositivo y pulsar F2 o DEL o la tecla correspondiente de tu dispositivo para entrar en la BIOS todo esto mientras se inicia.

En mi caso particular tuve que activar tanto Intel-VT-d y la virtualizacion.

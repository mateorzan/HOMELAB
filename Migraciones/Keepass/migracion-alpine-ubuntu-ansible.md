# Migración LXC Alpine → Ubuntu con Ansible Semaphore (Proxmox)

Documentación del proceso de automatización de la creación y configuración de una LXC Ubuntu en Proxmox usando Ansible Semaphore, como parte de la migración desde la LXC Alpine (servicio KeePass).

## Resumen

- **Nodo Proxmox**: `pve` (storage `local-zfs`)
- **Orquestador**: Ansible Semaphore (instalado en una LXC/VM dentro del mismo Proxmox)
- **Repositorio**: propio, gestionado por Git
- **Usuario final en la LXC**: `ubuntu-keepass`
- **Servicios a replicar**: Docker + Docker Compose, Tailscale (lo que corría en la Alpine)

---

## 1. Preparación en Proxmox

### 1.1. Usuario y rol de API dedicados (no root)

En vez de usar `root@pam`, se creó un usuario específico con permisos acotados:

1. **Datacenter → Permissions → Users → Add**
   - User: `ansible@pve`
2. **Datacenter → Permissions → Add → User Permission**
   - Path: `/` (importante: no `/vms` solo, porque el playbook también necesita acceso a `/storage/local` para leer la plantilla — un permiso limitado a `/vms` da un 403 al intentar usar el storage)
   - Role: `PVEVMAdmin` (rol predefinido, suficiente para crear/gestionar VMs y LXC)
3. **Datacenter → Permissions → API Tokens → Add**
   - User: `ansible@pve`
   - Token ID: `ansible-semaphore`
   - Privilege Separation: **desmarcado** (el token hereda los permisos del usuario directamente, sin necesitar una segunda asignación de rol al propio token)
   - Se copia el **Secret** generado (solo se muestra una vez)

### 1.2. Plantilla LXC de Ubuntu

Descargada desde la interfaz de Proxmox: **Storage `local` → CT Templates → Templates**, buscando "ubuntu" y descargando `ubuntu-24.04-standard` (versión `24.04-2`).

> Nota: una plantilla LXC (`.tar.zst`) **no** es lo mismo que una ISO de instalación (`.iso`). Los contenedores LXC no arrancan desde ISOs de instalador.

---

## 2. Configuración de Ansible Semaphore

### 2.1. Key Store

Se generó un par de claves SSH dedicado (no la clave personal):

```bash
ssh-keygen -t ed25519 -f ansible_semaphore_key -C "ansible@semaphore" -N ""
```

- La **clave privada** se guardó en Semaphore → **Key Store** (tipo SSH Key).
- La **clave pública** se usa como variable `lxc_pubkey` (ver más abajo) para inyectarla en la LXC al crearla.

### 2.2. Environment (`proxmox-creds`)

Todas las variables se definieron como **Environment Variables** (no Extra Variables), para que no aparezcan en texto plano en los logs de ejecución. Esto implica que en los playbooks se leen con `lookup('env', 'nombre_variable')` en vez de `{{ nombre_variable }}` directo.

| Variable                     | Tipo                                                                                                                        |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `proxmox_api_host`         | Environment Variable                                                                                                        |
| `proxmox_api_user`         | Environment Variable (formato:`ansible@pve`, **sin** el `!token_id`)                                              |
| `proxmox_api_token_id`     | Environment Variable (`ansible-semaphore`)                                                                                |
| `proxmox_api_token_secret` | Environment Variable                                                                                                        |
| `lxc_pubkey`               | Environment Variable (contenido completo de la clave pública)                                                              |
| `ubuntu_keepass_password`  | Environment Variable (contraseña del usuario`ubuntu-keepass`, en texto plano — Ansible la hashea con `password_hash`) |

### 2.3. Inventories

Se necesitaron **dos inventories distintos**, porque cada playbook se conecta de forma diferente:

**Inventory 1 — `local`** (para el playbook de creación, que solo llama a la API de Proxmox, sin SSH):

```ini
[local]
localhost ansible_connection=local
```

**Inventory 2 — `lxc-ubuntu-keepass`** (para el playbook de configuración, que sí se conecta por SSH a la IP real de la LXC):

```ini
[nuevas_lxc]
192.168.1.X ansible_user=root
```

(sustituir por la IP fija real asignada a la LXC; usa `ansible_user=root` porque el usuario `ubuntu-keepass` aún no existe hasta que corre el segundo playbook)

### 2.4. Dependencias del repositorio

**`requirements.yml`** (en la raíz del repo, se instala automáticamente):

```yaml
collections:
  - name: community.general
  - name: ansible.posix
```

**`requirements.txt`** (en la raíz del repo — **importante**: Semaphore NO instala automáticamente dependencias Python de un `requirements.txt` en el repo; solo aplica a `requirements.yml`. Hubo que instalar manualmente en el venv de Semaphore):

```
  tasks:
    - name: Instalar dependencias Python para el módulo de Proxmox
      ansible.builtin.pip:
        name:
          - proxmoxer
          - requests
        state: present
```

---

## 3. Playbook 1 — Creación de la LXC (`create_lxc.yml`)

```yaml
---
- name: Crear LXC Ubuntu en Proxmox
  hosts: localhost
  gather_facts: false
  vars:
    proxmox_node: "pve"
    ct_vmid: 200
    ct_hostname: "ubuntu-migrada"
    ct_ip: "192.168.1.X/24"
    ct_gateway: "192.168.1.1"
    ct_cores: 2
    ct_memory: 2048
    ct_disk_size: 20

  tasks:
    - name: Crear contenedor LXC
      community.general.proxmox:
        api_host: "{{ lookup('env', 'proxmox_api_host') }}"
        api_user: "{{ lookup('env', 'proxmox_api_user') }}"
        api_token_id: "{{ lookup('env', 'proxmox_api_token_id') }}"
        api_token_secret: "{{ lookup('env', 'proxmox_api_token_secret') }}"
        node: "{{ proxmox_node }}"
        vmid: "{{ ct_vmid }}"
        hostname: "{{ ct_hostname }}"
        ostemplate: "local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst"
        disk: "local-zfs:{{ ct_disk_size }}"
        cores: "{{ ct_cores }}"
        memory: "{{ ct_memory }}"
        netif: '{"net0":"name=eth0,bridge=vmbr0,ip={{ ct_ip }},gw={{ ct_gateway }}"}'
        unprivileged: true
        pubkey: "{{ lookup('env', 'lxc_pubkey') }}"
        state: present

    - name: Arrancar el contenedor
      community.general.proxmox:
        api_host: "{{ lookup('env', 'proxmox_api_host') }}"
        api_user: "{{ lookup('env', 'proxmox_api_user') }}"
        api_token_id: "{{ lookup('env', 'proxmox_api_token_id') }}"
        api_token_secret: "{{ lookup('env', 'proxmox_api_token_secret') }}"
        node: "{{ proxmox_node }}"
        vmid: "{{ ct_vmid }}"
        state: started
```

---

## 4. Playbook 2 — Configuración (`configure_lxc.yml`)

```yaml
---
- name: Configurar LXC Ubuntu (usuario, seguridad, Docker, Tailscale)
  hosts: nuevas_lxc
  become: true
  gather_facts: true
  vars:
    new_user: "ubuntu-keepass"
    user_pubkey: "{{ lookup('env', 'lxc_pubkey') }}"
    user_password: "{{ lookup('env', 'ubuntu_keepass_password') }}"

  tasks:
    - name: Actualizar cache de apt
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Actualizar todos los paquetes
      ansible.builtin.apt:
        upgrade: dist

    # --- Usuario ---
    - name: Crear usuario no-root
      ansible.builtin.user:
        name: "{{ new_user }}"
        shell: /bin/bash
        groups: sudo
        append: true
        create_home: true
        password: "{{ user_password | password_hash('sha512') }}"

    - name: Añadir clave pública SSH al nuevo usuario
      ansible.posix.authorized_key:
        user: "{{ new_user }}"
        key: "{{ user_pubkey }}"

    - name: Permitir autenticación SSH por contraseña
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication yes'
      notify: Reiniciar SSH

    # --- Seguridad base ---
    - name: Instalar ufw, fail2ban y unattended-upgrades
      ansible.builtin.apt:
        name:
          - ufw
          - fail2ban
          - unattended-upgrades
        state: present

    - name: Permitir SSH en ufw
      community.general.ufw:
        rule: allow
        port: '22'
        proto: tcp

    - name: Habilitar ufw
      community.general.ufw:
        state: enabled
        policy: deny

    - name: Habilitar unattended-upgrades
      ansible.builtin.copy:
        dest: /etc/apt/apt.conf.d/20auto-upgrades
        content: |
          APT::Periodic::Update-Package-Lists "1";
          APT::Periodic::Unattended-Upgrade "1";

    # --- Docker + Docker Compose ---
    - name: Instalar dependencias para el repo de Docker
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - gnupg
        state: present

    - name: Añadir clave GPG oficial de Docker
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc
        mode: '0644'
        force: true

    - name: Añadir repositorio de Docker
      ansible.builtin.apt_repository:
        repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present
        filename: docker

    - name: Instalar Docker Engine y Compose plugin
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-compose-plugin
        state: present
        update_cache: true

    - name: Añadir usuario al grupo docker
      ansible.builtin.user:
        name: "{{ new_user }}"
        groups: docker
        append: true

    # --- Tailscale ---
    - name: Instalar Tailscale (script oficial)
      ansible.builtin.shell: |
        curl -fsSL https://tailscale.com/install.sh | sh
      args:
        creates: /usr/bin/tailscale

  handlers:
    - name: Reiniciar SSH
      ansible.builtin.service:
        name: ssh
        state: restarted
```

---

## 5. Errores encontrados y solución (orden cronológico)

| Error | Causa | Solución |
| ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `'proxmox_api_host' is undefined` | Variables puestas como texto plano en el Environment, no accesibles como`{{ var }}` en Ansible sin más | Revisar tipo de variable en Semaphore (Environment Variable vs Extra Variable) |
| `'proxmox_api_token_secret' is undefined` (tras corregir lo anterior) | El secret no estaba correctamente guardado/confirmado en la sección Secrets del Environment | Verificar que la fila del secret esté realmente guardada en la tabla antes de hacer Update del Environment |
| `401 Unauthorized: Authentication failed` | `proxmox_api_user` llevaba el formato combinado `usuario!token_id`, pero el módulo espera el usuario y el token ID en campos separados | `proxmox_api_user` = solo `ansible@pve`; `proxmox_api_token_id` = solo `ansible-semaphore` |
| `403 Forbidden: Permission check failed (/storage/local, ...)` | El permiso del usuario`ansible@pve` estaba limitado al Path `/vms`, sin cubrir el storage | Cambiar el Path del permiso a`/` |
| `Failed to import the required Python library (proxmoxer)` | El módulo necesita el paquete Python`proxmoxer` en el venv de Ansible que usa Semaphore, y un `requirements.txt` en el repo **no se instala automáticamente** (solo aplica a `requirements.yml`) | Instalar manualmente:`/opt/semaphore/apps/ansible/<version>/venv/bin/pip install proxmoxer requests` |
| `500 Internal Server Error: Only root can pass arbitrary filesystem paths` | Sintaxis antigua del parámetro`disk` (número suelto + `storage` aparte), que las versiones nuevas del módulo/Proxmox ya no aceptan | Combinar en un solo parámetro:`disk: "local-zfs:{{ ct_disk_size }}"`, sin `storage:` aparte |

---

## 6. Pendiente / próximos pasos

- [ ] Migrar los datos/configuración de KeePass desde la LXC Alpine a la nueva LXC Ubuntu
- [ ] Autenticar Tailscale (`tailscale up`) — el playbook solo instala el binario, falta la conexión con auth key
- [ ] Decidir cómo exponer el puerto de KeePass en Docker: **Docker se salta las reglas de `ufw`** al publicar puertos, así que hay que publicar el puerto solo en la interfaz de Tailscale (o instalar `ufw-docker`) para no exponerlo sin querer a la LAN/internet
- [ ] Verificar funcionamiento completo antes de apagar/eliminar la LXC Alpine original

## 6. Rsync desde alpine a ubuntu

Vamos a copiar todos los datos de nuestros contenedores docker a la nueva LXC importante antes parar todos los servicios.

```
docker stop $(docker ps -q) # Para todos los contenedores en ejecución.

# Ahora copiamos con rsync
rsync -avz -e ssh \
  --exclude='.ssh' \
  --exclude='.ash_history' \
  --exclude='.local' \
  --exclude='.cache' \
  --exclude='.docker' \
  /root/ ubuntu-keepass@IP_NUEVA:/home/ubuntu-keepass/docker/

# Tambien tenemos que copiar los volumenes de docker con los que cree mis servicios.

rsync -avz -e ssh /var/lib/docker/volumes/ ubuntu-keepass@IP_NUEVA:/home/ubuntu-keepass/docker/volumes

# Luego los podemos arrancar
docker start $(docker ps -a -q)
```

Una vez copiado los volumenes hay que crearlos en la nueva maquina Ubuntu y moverlo a la carptea de docker con sudo

```
sudo su

cp -r /home/ubuntu-keepass/docker/volumes/<volume> /var/lib/docker/volumes/

docker volume create <volume-name>

docker volume list
```

## 7. Migracion de cada servicio

Ahora tenemos que migrar y volver a arrancar todos estos servicios.

- [X] n8n
- [X] vaultwarden
- [X] beszel
- [X] beszel-agent
- [X] uptime-kuma
- [X] Gotify
- [X] Portainer

### n8n

Hay que cambiar los permios de root a usuario.

```
sudo chown -R 1000:1000 /var/lib/docker/volumes/n8n_data

docker compose up -d y ya arranca el n8n
```

### Beszel

Beszel funciono sin problemas solo que aproveche y cree un compose ya que usaba docker run antes

```
services:
   beszel:
      image: henrygd/beszel
      container_name: beszel
      restart: unless-stopped
      environment:
        - APP_URL:http://localhost:8090
      ports:
        - 8090:8090
      volumes:
        - beszel_data:/beszel_data
volumes:
  beszel_data:
    external: true
```

### Gotify

Con gotify tuve un problema y es que no tenia ningun volumen ni carpeta asignada asi que tuve que sacar los datos del propio contenedor copiarlos a esta maquina ubuntu y crear el compose con el volumen asignado.

```
# Sacamos los datos del contenedor
docker exec gotify ls /app/data
docker cp gotify:/app/data ./gotify_data

# rsync para copiar los datos
rsync -avz -e ssh ./gotify_data ubuntu-keepass@192.168.1.X:/home/ubuntu-keepass/docker/gotify/

# docker-compose.yml
services:
   gotify:
      image: gotify/server
      container_name: gotify
      restart: unless-stopped
      ports:
        - 8081:80
      volumes:
        - ./gotify_data:/app/data
```

### Uptime Kuma

Con kuma me paso lo mismo que con n8n tuve problemas de permisos en el volumen y tuve que cambiarlos, ademas de crear el compose nuevo ya que lo lance con docker run en su momento.

```
sudo chown -R 1000:1000 /var/lib/docker/volumes/uptime-kuma

services:
   uptime-kuma:
       image: louislam/uptime-kuma:2
       container_name: uptime-kuma
       restart: always
       ports:
         - 3001:3001
       volumes:
         - uptime-kuma:/app/data
volumes:
   uptime-kuma:
     external: true
```

### Portainer

Con portainer no tuve problemas solo tuve que crear el compose.

```
services:
   portainer:
      image: portainer/portainer-ce:latest
      container_name: portainer
      restart: always
      ports:
        - 9000:9443
        - 8001:8001
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
        - portainer_data:/data
volumes:
   portainer_data:
      external: true
```

### Vaultwarden

En vaulwarden se me olvido copiar la carptea que guarda la data ya que no estaba en root estaba en /vw-data asi que tenemos que lanzar otro rsync.

```
# Copiamos la carpeta vw-data
rsync -avz -e ssh /vw-data/ ubuntu-keepass@IP:/home/ubuntu-keepass/docker/vaultwarden/vw-data/

# docker-compose.yml
services:
   vaultwarden:
      image: vaultwarden/server:latest
      container_name: vaultwarden
      restart: unless-stopped
      ports:
        - 8000:80
      environment:
        - DOMAIN=http://192.168.1.60:8000
      volumes:
        - ./vw-data/:/data/
```

### Upsnasp

Aqui no tuve que hacer nada ya tenia el compose solo lo levante

```
services:
  upsnap:
    container_name: upsnap
    image: ghcr.io/seriousm4x/upsnap:5
    restart: unless-stopped
    ports:
      - 8099:8099
    volumes:
      - ./data:/app/pb_data
    environment:                          # _ indentaci_n corregida (estaba con 5 espacios extra)
      - TZ=Europe/Madrid
      - UPSNAP_HTTP_LISTEN=0.0.0.0:8099  # _ cambiado de 127.0.0.1 a 0.0.0.0
      - UPSNAP_INTERVAL=*/10 * * * * *
      - UPSNAP_SCAN_RANGE=192.168.1.0/24
      - UPSNAP_SCAN_TIMEOUT=500ms
      - UPSNAP_PING_PRIVILEGED=true
      - UPSNAP_WEBSITE_TITLE=HOMELAB
    entrypoint: /bin/sh -c "apk update && apk add --no-cache openssh-client && rm -rf /var/cache/apk/* && ./upsnap serve"
```

### Besezel-agent

No tuve que hacer nada ya estaba el compose creado.

## 8. Cambio de IP (Opcional)

Debido a que esta es una migracion y la LXC de alpine se va a dejar de usar yo voy a restablecer la IP local que estaba usando asi mantengo las mismas urls y no tengo que cambiar mis accesos directos ni endpoint. Esto lo puedes hacer apagando las dos LXCs desde Proxmox y desde la LXC de Ubuntu vas a LXC-ubuntu --> Network --> net0.

Tambien por último nos quedaria iniciar sesion en Tailscale esto tambien lo deje para el final ya que sabia que iba a hacer este cambio de IP

`sudo tailscale up`

## Resultado

Ahora tenemos corriendo todo exactamente como lo teniamos en el mismo estado donde empezamos la migración. Objetivos conseguidos.

- No se perdio ningún dato.
- Todo sigue la misma configuración y funcionando igual.
- Mejoramos la seguridad y el mantenimiento de nuestro servidor.
- Ahora tenemos una distribucion mucho mas seguro y controlada aunque perdamos algo de rendimiento.

Hasta aquí la migración con esto aprendi sobre como trabajar con Ansible Semaphore para la gestion automática de LXCs dentro de proxmox, me parece algo muy útil y que voy a seguir implementando en todo mi Homelab.

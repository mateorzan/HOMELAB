# Configuración base de Seguridad Maquinas Homelab

Aqui voy a documentar todas las configuraciones basicas que tendrian que tener todos mis contenedroes para asegurarme de que sean seguros y den los menos errores posibles.

## Ubuntu

En toda maquina se necesitan dos cosas basicas controlar quien accede a nuestro servidor y mantener la ultima version para evitar brechas de seguridad, por ello es basico tener un firewall(ufw), un control de conexiones ssh(fail2ban) y actualizaciones automáticas(unattended-upgrades).

```
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar utilidades base
sudo apt install -y ufw fail2ban unattended-upgrades curl

# Firewall 
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.1.0/24 # Local 
sudo ufw allow from 100.64.0.0/10  # Tailscale
sudo ufw enable
sudo ufw status verbose

# fail2ban (bloqueo de fuerza bruta SSH)
sudo systemctl enable fail2ban --now
# Status servicio 
sudo systemctl status fail2ban
# Ver status de IPs baneadas ssh
sudo fail2ban-client status sshd 

# Actualizaciones automáticas de seguridad
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

* [X] Network-Services

  * [X] UFW
  * [X] Fail2Ban
  * [X] Unattend-Upgrades
* [X] GhostVM

  * [X] UFW
  * [X] Fail2Ban
  * [X] Unattend-Upgrades

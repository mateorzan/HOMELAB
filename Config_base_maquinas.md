
# Configuración base de Seguridad Maquinas Homelab

## Ubuntu

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

# Actualizaciones automáticas de seguridad
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

# Migración de Servicios — Proxmox Homelab

> **Objetivo:** Migrar los servicios de la VM CasaOS a dos contenedores LXC en el clúster Proxmox, distribuyendo la carga entre los dos nodos (ZimaBlade 1 y ZimaBlade 2).

---

## Infraestructura

| Elemento               | Detalle                      |
| ---------------------- | ---------------------------- |
| **Nodo 1**       | ZimaBlade1 — IP:`x.x.x.x` |
| **Nodo 2**       | ZimaBlade2 — IP:`x.x.x.x` |
| **Storage LXCs** | `LXC_Servicios`(LVM-thin)  |
| **VM origen**    | CasaOS — ID:`xxx`         |
| **Datos origen** | `/DATA/AppData/`           |

---

## Distribución de servicios

### LXC 1 — Servicios de red (Nodo 1)

> Servicios críticos para el acceso y la conectividad. Replicado en Nodo 2.

| Servicio            | Puerto      | Estado       |
| ------------------- | ----------- | ------------ |
| Nginx Proxy Manager | 80, 443, 81 | ⏳ Pendiente |
| ddns-updater        | -           | ⏳ Pendiente |
| Sure                | -           | ⏳ Pendiente |

* **LXC ID:**
* **IP asignada:**
* **RAM asignada:**
* **Replicación:** ✅ Sí — cada `__` minutos → Nodo 2

---

### LXC 2 — Servicios de aplicaciones (Nodo 2)

> Servicios de uso diario. Sin replicación por volumen de datos.

| Servicio  | Puerto | Estado       |
| --------- | ------ | ------------ |
| Jellyfin  | 8096   | ⏳ Pendiente |
| Nextcloud | -      | ⏳ Pendiente |
| Deluge    | -      | ⏳ Pendiente |
| Radarr    | 7878   | ⏳ Pendiente |
| Sonarr    | 8989   | ⏳ Pendiente |
| Prowlarr  | 9696   | ⏳ Pendiente |
| Syncthing | -      | ⏳ Pendiente |
| Portainer | 9000   | ⏳ Pendiente |

* **LXC ID:**
* **IP asignada:**
* **RAM asignada:**
* **Replicación:** ❌ No — demasiado volumen de datos

---

## Registro de migración

### [ ] LXC 1 — Servicios de red

* **Fecha:**
* **Datos copiados desde:**
  * `/DATA/AppData/nginx-proxy-manager/`
  * `/DATA/AppData/ddns-updater/`
* **Verificaciones:**
  * [ ] Nginx resuelve todos los dominios correctamente
  * [ ] DDNS actualiza la IP externa
  * [ ] Replicación hacia Nodo 2 activa
* **Notas:**

---

### [ ] LXC 2 — Servicios de aplicaciones

* **Fecha:**
* **Datos copiados desde:**
  * `/DATA/AppData/jellyfin/`
  * `/DATA/AppData/nextcloud/`
  * `/DATA/AppData/deluge/`
  * `/DATA/AppData/radarr/`
  * `/DATA/AppData/sonarr/`
  * `/DATA/AppData/prowlarr/`
  * `/DATA/AppData/syncthing/`
* **Ruta de media (Jellyfin/Radarr/Sonarr):**
* **Hardware transcoding Jellyfin:** ⏳ Por configurar (Intel QuickSync)
* **Cron Nextcloud:** cada `__` minutos
* **Verificaciones:**
  * [ ] Jellyfin reproduce sin transcodificación por software
  * [ ] Nextcloud accesible y sincronizando
  * [ ] Radarr/Sonarr conectados a Prowlarr y Deluge
* **Notas:**

---

## Checklist general por LXC

```
[ ] LXC creado en Proxmox con IP fija
[ ] Datos copiados desde CasaOS
[ ] Permisos ajustados (chown)
[ ] Servicio arranca correctamente
[ ] Nginx Proxy Manager apunta al nuevo LXC
[ ] Contenedor antiguo en CasaOS apagado (no borrado)
[ ] Replicación configurada si aplica
[ ] Snapshot inicial hecho
[ ] Contenedor antiguo borrado (tras periodo de prueba)
```

---

## Comandos útiles

```bash
# Copiar datos desde CasaOS al LXC nuevo
scp -r /DATA/AppData/servicio/ usuario@IP_LXC:/ruta/destino/

# Ver estado de los LXCs
pct list
pct status <ID>

# Snapshot manual antes de cambios
pct snapshot <ID> pre-migracion --description "Antes de migrar servicio X"

# Arrancar/parar LXC
pct start <ID>
pct stop <ID>

# Acceder a la consola de un LXC
pct enter <ID>

# Borrar VM con lock
qm unlock <VMID>
qm destroy <VMID> --purge
```

---

## Notas generales

* Dejar la VM de CasaOS apagada pero sin borrar hasta confirmar que todo funciona
* Usar IPs fijas en el mismo rango que el resto de la red
* Documentar cualquier cambio de puerto respecto a la configuración anterior
* El LXC 1 es prioritario — migrarlo primero ya que Nginx es la puerta de entrada a todo

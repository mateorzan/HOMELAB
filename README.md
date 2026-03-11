# 🏠 HOMELAB INFRASTRUCTURE PROJECT

### 🚀 High Availability Cluster with ZimaBlade & Raspberry Pi

<p align="center">

[![Virtualization](https://img.shields.io/badge/Virtualization-Proxmox%20VE-E57000?style=for-the-badge&logo=proxmox&logoColor=white)](https://www.proxmox.com/en/proxmox-virtual-environment)
[![Containers](https://img.shields.io/badge/Containers-Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![VPN](https://img.shields.io/badge/Network-Tailscale-000000?style=for-the-badge&logo=tailscale&logoColor=white)](https://tailscale.com/)
[![Backups](https://img.shields.io/badge/Backups-Proxmox%20Backup%20Server-009688?style=for-the-badge&logo=proxmox&logoColor=white)](https://www.proxmox.com/en/proxmox-backup-server)
[![ARM Node](https://img.shields.io/badge/ARM-Raspberry%20Pi%205-C51A4A?style=for-the-badge&logo=raspberrypi&logoColor=white)](https://www.raspberrypi.com/products/raspberry-pi-5/)
[![Status](https://img.shields.io/badge/Status-Active%20Development-brightgreen?style=for-the-badge)](#)

</p>

---

# 📖 Descripción

En este repositorio voy a explicar como yo configure mi actual homelab, compartire algunos archivos de configuracion que pueden ser de ayuda para la gente que quiere hacer algo parecido en su propio servidor.

Este proyecto tiene como objetivo construir una infraestructura doméstica orientada a:

- 🏗️ Alta disponibilidad
- 🔁 Replicación entre nodos
- 💾 Backups incrementales diarios
- ⚖️ Balance estructural
- 🌍 Servicios 24/7
- 🤖 Nodo independiente para IA

---

# 🏗️ Arquitectura General

## 🔹 Diagrama de Infraestructura

```mermaid
graph TD

    Router --> Tailscale
    Tailscale --> ProxmoxCluster
    Tailscale --> RaspberryPi

    subgraph Datacenter
        ProxmoxCluster --> Nodo1[ZimaBlade1 PVE]
        ProxmoxCluster --> Nodo2[ZimaBlade2 PVE2]
        Nodo1 --> CasaOS
        Nodo2 --> PBS[Backup Server]
	Nodo2 --> CT_Keepas
    end

    RaspberryPi --> IA[Servicios IA]
```

---

## 🔹 Estructura Actual

```
Datacenter (Proxmox Cluster)
│
├── pve     → zimablade1 (Nodo 1 - Servicios)
├── pve2    → zimablade2 (Nodo 2 - Backup / Replicacion / Servicios)
│
└── Raspberry Pi 5
    └── Servidor independiente corriendo servicios de IA
```

---

## 🎯 Estructura Objetivo

```
Datacenter (Alta Disponibilidad)
│
├── Nodo 1 → Servicios principales
├── Nodo 2 → Replicación + Backups
├── Nodo 3 → Expansión futura (Quorum)
│
└── Raspberry Pi 5 → Laboratorio IA (ARM)
```

---

# 🧠 Arquitectura Técnica

## 🔹 Virtualización

- Cluster basado en Proxmox VE
- Contenedores LXC para servicios ligeros
- Máquinas virtuales para servicios críticos
- Snapshots periódicos

## 🔹 Backups

- Backups incrementales diarios
- Deduplicación
- Compresión
- Replicación entre nodos

## 🔹 Red

- VPN Mesh privada
- Acceso remoto seguro
- Servicios expuestos mediante proxy inverso
- Certificados SSL

## 🔹 Orquestación

- Gestión manual estructurada
- Evaluación futura de Kubernetes
- Segmentación por tipo de servicio

---

# 📊 Servicios Activos

## 🔹 Infraestructura

| Servicio              | Función              |
| --------------------- | --------------------- |
| Proxmox VE            | Virtualización       |
| Proxmox Backup Server | Backups               |
| CasaOS                | Gestión de servicios |
| Tailscale             | VPN privada           |
| Portainer             | Gestión Docker       |
| Beszel                | Monitorización       |
| Uptime Kuma           | Monitorización       |
| UpSnap                | WakeOnLAN             |

---

## 🔹 Servicios Usuario

| Servicio            | Función                   |
| ------------------- | -------------------------- |
| NextCloud           | Almacenamiento Cloud Local |
| Jellyfin            | Servicio Multimedia        |
| Nginx Proxy Manager | Proxy inverso              |
| Radarr              | Gestión Películas        |
| Sonar               | Gestión Series            |
| Prowlar             | Indexer Torrent            |
| Deluge              | Cliente Torrent            |
| Downtify            | Música                    |
| Sure                | Gestión de gastos         |
| Ddns-Updater        | Actualización IP pública |
| VaultWarden         | Gestor de contraseñas     |
| Gotify              | Chat de alertas            |
| N8N                 | Automatizacion             |

---

# 📈 Métricas de Infraestructura

*(Sección preparada para actualizar con métricas reales)*

- 🖥️ Nodos activos: 2
- 💾 Tipo almacenamiento: HDD
- 🌐 Acceso remoto: VPN privada
- 🔄 Backups: Incrementales diarios
- ⏱️ Objetivo disponibilidad: 24/7

---

# 🛣️ Roadmap Técnico

## Infraestructura

- [X] Migración a Proxmox
- [X] Implementación de PBS
- [X] Integración VPN privada
- [X] Monitorización avanzada
- [ ] Añadir nodo para quorum
- [ ] Migración a almacenamiento SSD

## Automatización

- [X] Backups automáticos verificados
- [X] Alertas por caída de servicios
- [ ] CI/CD para despliegues

## IA (Raspberry Pi 5)

- [ ] Servidor IA local Asistente
- [ ] Automatización inteligente
- [ ] Experimentos ML

---

# 🧪 Evolución del Proyecto

## 🏗 Fase Inicial

Una raspberry Pi5 con CasaOS instalado localmente con diferentes servicios creados a traves de CasaOS:

- NextCloud
- Jellyfin
- Nginx Proxy Manager
- Prowlar
- Sonar
- Radarr
- DdnsUpdater
- Sure

---

# 🎯 Objetivo Final

Crear una red de nodos de alta disponibilidad y respaldada, este proyecto es muy ambicioso y aun no se el alcance de este mismo, probablemente ira a mas, ire actualizando con las novedades este Readme.

Este proyecto funciona como:

- 🧠 Laboratorio personal de infraestructura
- 🏗️ Entorno de pruebas DevOps
- 🔬 Plataforma de experimentación IA
- 📚 Proyecto documentado como portfolio técnico

---

# 👨‍💻 Autor

Proyecto desarrollado como laboratorio personal de infraestructura, virtualización y servicios autoalojados.

Infraestructura en evolución constante 🚀

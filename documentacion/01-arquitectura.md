# 01 - Arquitectura de Red

## 1.1 Introducción

Este documento describe la arquitectura de red diseñada e implementada para la empresa TechLogix, una compañía dedicada a la gestión logística que requiere una infraestructura moderna, segura y escalable.

La arquitectura sigue un modelo de defensa en profundidad con múltiples capas de seguridad, segmentación mediante VLANs, y servicios centralizados para facilitar la administración.

---

## 1.2 Topología de Red

### 1.2.1 Diseño General

Se ha implementado una **topología en estrella** con pfSense como núcleo central de enrutamiento y seguridad. Esta topología facilita:

- Mantenimiento centralizado
- Escalabilidad horizontal
- Aislamiento de fallos
- Control de tráfico entre segmentos

### 1.2.2 Diagrama de Red

```
                                          ┌─────────────────┐
                                          │     INTERNET    │
                                          │   (NAT Network) │
                                          └────────┬────────┘
                                                   │
                                                   │ WAN: 10.10.10.4
                                          ┌────────┴────────┐
                                          │    pfSense      │
                                          │   CE 2.7.x      │
                                          │                 │
                                          │  • Firewall     │
                                          │  • Router       │
                                          │  • Snort IDS    │
                                          │  • DHCP Relay   │
                                          └────────┬────────┘
                                                   │
       ┌───────────────┬───────────────┬───────────┼───────────┬───────────────┬───────────────┐
       │               │               │           │           │               │               │
  ┌────┴────┐    ┌─────┴─────┐   ┌─────┴─────┐ ┌───┴───┐ ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
  │ VLAN 10 │    │  VLAN 20  │   │  VLAN 30  │ │VLAN 40│ │  VLAN 50  │   │  VLAN 60  │   │  VLAN 70  │
  │  ADMIN  │    │   PROD    │   │    IT     │ │SERVERS│ │WIFI GUEST │   │ SECURITY  │   │MONITORING │
  │ .10.0/24│    │  .20.0/24 │   │  .30.0/24 │ │.40.0/24│ │  .50.0/24 │   │  .60.0/24 │   │  .70.0/24 │
  └────┬────┘    └─────┬─────┘   └─────┬─────┘ └───┬───┘ └───────────┘   └─────┬─────┘   └─────┬─────┘
       │               │               │           │                           │               │
  ┌────┴────┐    ┌─────┴─────┐   ┌─────┴─────┐ ┌───┴───┐                   ┌────┴────┐    ┌────┴────┐
  │CLI-WIN  │    │  Equipos  │   │ CLI-IT    │ │ DC01  │                   │ SEC01   │    │ MON01   │
  │ ADMIN   │    │   Prod    │   │ Debian    │ │ DC02  │                   │ OpenVPN │    │ Zabbix  │
  │         │    │           │   │           │ │FILE01 │                   │         │    │ Grafana │
  └─────────┘    └───────────┘   └───────────┘ │BAK01  │                   └─────────┘    │ Ansible │
                                               └───┬───┘                                  └─────────┘
                                                   │
                                              ┌────┴────┐
                                              │VLAN 100 │
                                              │   DMZ   │
                                              │.100.0/24│
                                              └────┬────┘
                                                   │
                                              ┌────┴────┐
                                              │ WEB01   │
                                              │WordPress│
                                              └─────────┘
```

---

## 1.3 Segmentación de VLANs

### 1.3.1 Tabla de VLANs

| VLAN ID | Nombre | Subred | Gateway | Propósito |
|---------|--------|--------|---------|-----------|
| 10 | ADMIN | 192.168.10.0/24 | 192.168.10.1 | Administradores de sistemas |
| 20 | PROD | 192.168.20.0/24 | 192.168.20.1 | Departamento de producción |
| 30 | IT | 192.168.30.0/24 | 192.168.30.1 | Departamento técnico |
| 40 | SERVERS | 192.168.40.0/24 | 192.168.40.1 | Servidores internos |
| 50 | WIFI_GUESTS | 192.168.50.0/24 | 192.168.50.1 | Red WiFi invitados |
| 60 | SECURITY | 192.168.60.0/24 | 192.168.60.1 | VPN y seguridad |
| 70 | MONITORING | 192.168.70.0/24 | 192.168.70.1 | Monitorización |
| 100 | DMZ | 192.168.100.0/24 | 192.168.100.1 | Zona desmilitarizada |

### 1.3.2 Justificación de la Segmentación

**VLAN 10 - ADMIN:** 
Aislamiento de equipos de administradores con acceso privilegiado a todos los recursos internos excepto DMZ y WiFi Guests.

**VLAN 20 - PROD:**
Departamento de producción con acceso limitado a servidor de archivos (SMB) y servicios de dominio.

**VLAN 30 - IT:**
Personal técnico con acceso a servidores vía SSH, SMB, y herramientas de monitorización.

**VLAN 40 - SERVERS:**
Red de servidores internos protegida. Solo accesible desde VLANs autorizadas en puertos específicos.

**VLAN 50 - WIFI_GUESTS:**
Red completamente aislada para invitados. Solo permite acceso a Internet (HTTP/HTTPS y DNS). Bloqueada de toda red interna.

**VLAN 60 - SECURITY:**
Servicios de seguridad (OpenVPN). Acceso controlado hacia recursos internos.

**VLAN 70 - MONITORING:**
Servidor de monitorización con acceso de sondeo (polling) a todos los servidores.

**VLAN 100 - DMZ:**
Zona desmilitarizada para servicios públicos (web). Aislada de la red interna.

---

## 1.4 Direccionamiento IP

### 1.4.1 Servidores

| Servidor | Hostname | IP | VLAN | MAC Address |
|----------|----------|-----|------|-------------|
| Controlador Dominio 1 | SRV-DC01 | 192.168.40.10 | 40 | Dinámica VBox |
| Controlador Dominio 2 | SRV-DC02 | 192.168.40.14 | 40 | Dinámica VBox |
| Servidor Archivos | SRV-FILE01 | 192.168.40.12 | 40 | Dinámica VBox |
| Servidor Backup | SRV-BAK01 | 192.168.40.13 | 40 | Dinámica VBox |
| Servidor Web | SRV-WEB01 | 192.168.100.8 | 100 | Dinámica VBox |
| Servidor Seguridad | SRV-SEC01 | 192.168.60.10 | 60 | Dinámica VBox |
| Servidor Monitorización | SRV-MON01 | 192.168.70.10 | 70 | Dinámica VBox |

### 1.4.2 Clientes

| Cliente | Hostname | IP | VLAN |
|---------|----------|-----|------|
| Windows Admin | CLI-WIN-ADMIN | DHCP (192.168.10.x) | 10 |
| Debian IT | CLI-IT-DEBIAN | DHCP (192.168.30.x) | 30 |
| VPN Client | CLI-DEBIAN-VPN | 10.8.0.x (túnel) | Externa |

### 1.4.3 Equipos de Red

| Equipo | IP | Interfaz | Función |
|--------|-----|----------|---------|
| pfSense WAN | 10.10.10.4 | em0 | Conexión Internet |
| pfSense LAN | 192.168.40.1 | em1 | Gateway VLAN Servers |
| pfSense OPT1 | 192.168.100.1 | em1.100 | Gateway DMZ |
| pfSense OPT2 | 192.168.60.1 | em1.60 | Gateway Security |
| pfSense OPT3 | 192.168.70.1 | em1.70 | Gateway Monitoring |
| pfSense OPT4 | 192.168.10.1 | em1.10 | Gateway Admin |
| pfSense OPT5 | 192.168.20.1 | em1.20 | Gateway Prod |
| pfSense OPT6 | 192.168.30.1 | em1.30 | Gateway IT |

---

## 1.5 Entorno de Virtualización

### 1.5.1 Plataforma

- **Hipervisor:** Oracle VirtualBox 7.0
- **Host:** Windows/Linux con 32GB RAM
- **Almacenamiento:** SSD 500GB+

### 1.5.2 Configuración de Red en VirtualBox

Todas las máquinas virtuales utilizan **Redes Internas (Internal Network)** conectadas a pfSense para enrutamiento entre VLANs.

| Red Interna | VLAN | Máquinas Conectadas |
|-------------|------|---------------------|
| LAN | 40 | DC01, DC02, FILE01, BAK01, pfSense |
| DMZ | 100 | WEB01, pfSense |
| SECURITY | 60 | SEC01, pfSense |
| MONITORING | 70 | MON01, pfSense |
| ADMIN | 10 | CLI-WIN-ADMIN, pfSense |
| INTERNET | WAN | pfSense, CLI-DEBIAN-VPN |

### 1.5.3 Recursos por Máquina Virtual

| Servidor | RAM | CPU | Disco | Notas |
|----------|-----|-----|-------|-------|
| pfSense | 1 GB | 1 | 20 GB | FreeBSD |
| DC01 | 2 GB | 2 | 40 GB | Samba AD |
| DC02 | 2 GB | 2 | 40 GB | Samba AD réplica |
| FILE01 | 1 GB | 1 | 40 GB | SMB + FTP |
| BAK01 | 2 GB | 2 | 40 GB + RAID 5 | Bacula |
| WEB01 | 1 GB | 1 | 40 GB | LAMP + WordPress |
| SEC01 | 1 GB | 1 | 20 GB | OpenVPN |
| MON01 | 2 GB | 2 | 40 GB | Zabbix + Grafana |

---

## 1.6 Flujo de Tráfico

### 1.6.1 Tráfico Norte-Sur (Internet)

```
Cliente Interno → pfSense (NAT) → Internet
                      ↓
              Snort IDS analiza
                      ↓
              Reglas Firewall
```

### 1.6.2 Tráfico Este-Oeste (Entre VLANs)

```
VLAN Admin → pfSense → Reglas Firewall → VLAN Servers
                            ↓
                    Solo puertos permitidos
                    (SSH, SMB, RDP, etc.)
```

### 1.6.3 Tráfico VPN

```
Usuario Remoto → Internet → pfSense:1194 → NAT → SRV-SEC01:1194
                                                      ↓
                                               Túnel OpenVPN
                                                      ↓
                                               IP: 10.8.0.x
                                                      ↓
                                               Acceso a FILE01:21
```

---

## 1.7 Alta Disponibilidad

### 1.7.1 Servicios Redundantes

| Servicio | Primario | Secundario | Tipo |
|----------|----------|------------|------|
| Active Directory | DC01 | DC02 | Replicación Samba |
| DNS | DC01 | DC02 | Integrado con AD |
| Almacenamiento Backup | RAID 5 | Google Drive | 3-2-1 Strategy |

### 1.7.2 Consideraciones Futuras

Para un entorno de producción real se recomienda:
- CARP/VRRP para redundancia de pfSense
- Cluster de almacenamiento (GlusterFS, Ceph)
- Balanceador de carga para servicios web

---

## 1.8 Conclusiones

La arquitectura implementada cumple con los requisitos de:

- ✅ **Segmentación:** 8 VLANs con propósitos diferenciados
- ✅ **Seguridad:** Firewall perimetral + IDS + firewalls host-based
- ✅ **Escalabilidad:** Diseño modular permite agregar VLANs y servidores
- ✅ **Administración:** Gestión centralizada desde VLAN de monitorización
- ✅ **Acceso remoto:** VPN con autenticación contra Active Directory

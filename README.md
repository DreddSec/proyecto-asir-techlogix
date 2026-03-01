# 🏢 TechLogix - Infraestructura de Red Empresarial

<p align="center">
  <img src="https://img.shields.io/badge/Estado-Completado-brightgreen" alt="Estado">
  <img src="https://img.shields.io/badge/Plataforma-Ubuntu%20Server%2024.04-orange" alt="Plataforma">
  <img src="https://img.shields.io/badge/Virtualizaci%C3%B3n-VirtualBox-blue" alt="Virtualización">
  <img src="https://img.shields.io/badge/Firewall-pfSense-red" alt="Firewall">
  <img src="https://img.shields.io/badge/Proyecto-ASIR-purple" alt="ASIR">
</p>


## 📋 Descripción

Proyecto final del ciclo formativo **ASIR (Administración de Sistemas Informáticos en Red)** que implementa una infraestructura de red completa para la empresa ficticia **TechLogix**, dedicada a la gestión logística.

El proyecto abarca el diseño, implementación y documentación de una red empresarial que incluye:

- Segmentación de red mediante VLANs
- Servicios de directorio con Active Directory (Samba AD)
- Servidor de archivos con SMB y FTP
- Acceso remoto seguro mediante VPN
- Sistema de backup con estrategia 3-2-1
- Monitorización centralizada
- Sistema de detección de intrusiones (IDS)

---

## 🎯 Objetivos del Proyecto

| Objetivo                                          | Estado |
| ------------------------------------------------- | ------ |
| Diseñar una arquitectura de red robusta           | ✅      |
| Implementar un sistema de gestión de datos seguro | ✅      |
| Establecer políticas de seguridad                 | ✅      |
| Optimizar el rendimiento de la red                | ✅      |
| Implementar sistema de respaldo 3-2-1             | ✅      |
| Configurar acceso remoto VPN                      | ✅      |
| Monitorización de infraestructura                 | ✅      |

---

## 🏗️ Arquitectura de Red

### Diagrama de Topología

```
                                    ┌─────────────┐
                                    │   INTERNET  │
                                    └──────┬──────┘
                                           │
                                    ┌──────┴──────┐
                                    │   pfSense   │
                                    │  (Firewall) │
                                    └──────┬──────┘
                                           │
            ┌──────────────────────────────┼──────────────────────────────┐
            │                              │                              │
    ┌───────┴───────┐              ┌───────┴───────┐              ┌───────┴───────┐
    │   VLAN 40     │              │   VLAN 100    │              │   VLAN 60     │
    │   SERVERS     │              │     DMZ       │              │   SECURITY    │
    │ 192.168.40.0  │              │ 192.168.100.0 │              │ 192.168.60.0  │
    └───────┬───────┘              └───────┬───────┘              └───────┬───────┘
            │                              │                              │
    ┌───────┴───────┐              ┌───────┴───────┐              ┌───────┴───────┐
    │ • DC01        │              │ • SRV-WEB01   │              │ • SRV-SEC01   │
    │ • DC02        │              │   (WordPress) │              │   (OpenVPN)   │
    │ • SRV-FILE01  │              │               │              │   (Snort)     │
    │ • SRV-BAK01   │              │               │              │               │
    └───────────────┘              └───────────────┘              └───────────────┘

    ┌───────────────┐              ┌───────────────┐              ┌───────────────┐
    │   VLAN 70     │              │   VLAN 10     │              │   VLAN 50     │
    │  MONITORING   │              │    ADMIN      │              │  WIFI GUESTS  │
    │ 192.168.70.0  │              │ 192.168.10.0  │              │ 192.168.50.0  │
    └───────┬───────┘              └───────┬───────┘              └───────────────┘
            │                              │
    ┌───────┴───────┐              ┌───────┴───────┐
    │ • SRV-MON01   │              │ • CLI-WIN-ADMIN│
    │   (Zabbix)    │              │   (Windows 11)│
    │   (Grafana)   │              │               │
    └───────────────┘              └───────────────┘
```

### Segmentación de VLANs

| VLAN ID | Nombre      | Subred           | Propósito                   |
| ------- | ----------- | ---------------- | --------------------------- |
| 10      | ADMIN       | 192.168.10.0/24  | Administradores de sistemas |
| 20      | PROD        | 192.168.20.0/24  | Departamento de producción  |
| 30      | IT          | 192.168.30.0/24  | Departamento de IT          |
| 40      | SERVERS     | 192.168.40.0/24  | Servidores internos         |
| 50      | WIFI_GUESTS | 192.168.50.0/24  | Red WiFi para invitados     |
| 60      | SECURITY    | 192.168.60.0/24  | VPN e IDS                   |
| 70      | MONITORING  | 192.168.70.0/24  | Monitorización              |
| 100     | DMZ         | 192.168.100.0/24 | Zona desmilitarizada        |

---

## 🖥️ Inventario de Servidores

| Servidor     | IP               | Sistema Operativo   | Servicios                   |
| ------------ | ---------------- | ------------------- | --------------------------- |
| SRV-DC01     | 192.168.40.10    | Ubuntu Server 24.04 | Samba AD, DNS, DHCP         |
| SRV-DC02     | 192.168.40.14    | Ubuntu Server 24.04 | Samba AD (réplica), DNS     |
| SRV-FILE01   | 192.168.40.12    | Ubuntu Server 24.04 | SMB, FTP (vsftpd)           |
| SRV-BAK01    | 192.168.40.13    | Ubuntu Server 24.04 | Bacula, RAID 5, Rclone      |
| SRV-WEB01    | 192.168.100.8    | Ubuntu Server 24.04 | Apache, WordPress, MySQL    |
| SRV-SEC01    | 192.168.60.10    | Ubuntu Server 24.04 | OpenVPN                     |
| SRV-MON01    | 192.168.70.10    | Ubuntu Server 24.04 | Zabbix, Grafana, Ansible    |
| FW-PFSENSE01 | 10.10.10.4 (WAN) | FreeBSD (pfSense)   | Firewall, Router, Snort IDS |

---

## 🔧 Servicios Implementados

### 🔐 Active Directory (Samba AD)

- **Dominio:** TECHLOGIX.LOCAL

- **Controladores:** DC01 (primario) + DC02 (réplica)

- **Estructura organizativa:**

  ```
  TECHLOGIX.LOCAL
  ├── OU=Usuarios
  │   ├── OU=Administracion
  │   ├── OU=IT
  │   └── OU=Produccion
  └── OU=Grupos
      ├── GRP_Administracion
      ├── GRP_IT
      └── GRP_Produccion
  ```

### 📁 Servidor de Archivos (SMB + FTP)

**Carpetas compartidas SMB:**

| Recurso        | Ruta                           | Grupo con acceso   |
| -------------- | ------------------------------ | ------------------ |
| Comun          | /srv/compartido/comun          | Domain Users       |
| Administracion | /srv/compartido/administracion | GRP_Administracion |
| IT             | /srv/compartido/it             | GRP_IT             |
| Produccion     | /srv/compartido/produccion     | GRP_Produccion     |

**FTP:**

- Servidor vsftpd para transferencias externas
- Acceso mediante usuarios locales
- Integrado con acceso VPN para trabajo remoto

### 🔒 Seguridad

#### VPN (OpenVPN)

- Puerto: 1194/UDP
- Autenticación: Certificados + LDAP contra Samba AD
- Acceso restringido al grupo GRP_IT
- Cifrado: AES-256-GCM

#### Firewall (pfSense)

- Reglas granulares por VLAN
- Principio de mínimo privilegio
- Port forwarding para servicios públicos
- NAT para acceso a Internet

#### IDS (Snort)

- Instalado en pfSense
- Interfaces monitorizadas: WAN, LAN, DMZ, VPN
- Reglas VRT + Community
- Modo IPS: Bloqueo automático de amenazas

#### UFW (Host-based Firewall)

- Configurado en todos los servidores Ubuntu
- Segunda capa de defensa
- Reglas específicas por servicio

### 💾 Sistema de Backup (Estrategia 3-2-1)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   ORIGINAL  │     │   LOCAL     │     │    CLOUD    │
│   (Datos)   │ ──► │  (RAID 5)   │ ──► │  (GDrive)   │
│             │     │  SRV-BAK01  │     │   Rclone    │
└─────────────┘     └─────────────┘     └─────────────┘
      1                   2                   3
   Copia              Copias              Ubicación
  Original           Locales              Remota
```

- **Bacula Director:** Orquesta backups de todos los servidores
- **RAID 5:** 5 discos de 40GB para redundancia local
- **Rclone:** Sincronización automática a Google Drive (cron 3:00 AM)
- **Datos respaldados:** /etc, /home, /var/log, /var/www

### 📊 Monitorización

#### Zabbix

- Agentes instalados en todos los servidores
- Monitorización de: CPU, RAM, Disco, Red, Servicios
- Alertas por email configuradas

#### Grafana

- Datasource: Zabbix
- Dashboards personalizados:
  - Visión general de infraestructura
  - Métricas de rendimiento
  - Estado de servicios

---

## 🛡️ Políticas de Seguridad

### Segmentación de Red

| Origen            | Destino    | Acceso                      |
| ----------------- | ---------- | --------------------------- |
| ADMIN             | SERVERS    | ✅ Total                     |
| ADMIN             | DMZ        | ❌ Bloqueado                 |
| ADMIN             | MONITORING | ✅ Zabbix/Grafana            |
| PROD              | SERVERS    | ✅ Solo SMB (File Server)    |
| IT                | SERVERS    | ✅ SSH + SMB                 |
| IT                | MONITORING | ✅ Zabbix/Grafana            |
| DMZ               | SERVERS    | ❌ Bloqueado                 |
| WIFI_GUESTS       | Interno    | ❌ Bloqueado (solo Internet) |
| VPN (10.8.0.0/24) | FILE01     | ✅ Solo FTP                  |

### Hardening de Servidores

- SSH en puerto personalizado (2222)
- Acceso SSH solo con clave pública
- Fail2ban instalado
- UFW configurado con reglas mínimas
- Actualizaciones automáticas de seguridad

---

## 📂 Estructura del Repositorio

```
proyecto-asir-techlogix/
│
├── README.md                    # Este archivo
├── LICENSE
│
├── documentacion/
│   ├── 01-arquitectura.md       # Diseño de red
│   ├── 02-servicios.md          # DNS, DHCP, SMB, FTP
│   ├── 03-seguridad.md          # VPN, Firewall, IDS
│   ├── 04-backup.md             # Bacula + Rclone
│   ├── 05-monitorizacion.md     # Zabbix + Grafana
│   └── memoria-proyecto.pdf     # Documento formal
│
├── configuraciones/
│   ├── samba-ad/                # smb.conf, krb5.conf
│   ├── openvpn/                 # server.conf, auth-ldap.conf
│   ├── bacula/                  # bacula-dir.conf, etc.
│   └── firewall/                # Reglas pfSense + UFW
│
├── automatizaciones/
│   ├── scripts/                 # Scripts de mantenimiento
│   └── ansible/                 # Playbooks de configuración
│
└── diagramas/
    ├── topologia-red.png        # Diagrama de red
    └── packet-tracer/           # Archivos .pkt
```

---

## 🚀 Tecnologías Utilizadas

<p align="center">
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/pfSense-003366?style=for-the-badge&logo=pfsense&logoColor=white" alt="pfSense">
  <img src="https://img.shields.io/badge/Samba-FCC624?style=for-the-badge&logo=samba&logoColor=black" alt="Samba">
  <img src="https://img.shields.io/badge/OpenVPN-EA7E20?style=for-the-badge&logo=openvpn&logoColor=white" alt="OpenVPN">
  <img src="https://img.shields.io/badge/Zabbix-D50000?style=for-the-badge&logo=zabbix&logoColor=white" alt="Zabbix">
  <img src="https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white" alt="Grafana">
  <img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white" alt="Ansible">
  <img src="https://img.shields.io/badge/VirtualBox-183A61?style=for-the-badge&logo=virtualbox&logoColor=white" alt="VirtualBox">
</p>


---

## 📝 Requisitos del Entorno

### Software

- VirtualBox 7.0+
- Packet Tracer (para diseño de red)
- Git

---

## 📖 Documentación Adicional

- [Arquitectura de Red](documentacion/01-arquitectura.md)
- [Servicios de Red](documentacion/02-servicios.md)
- [Seguridad](documentacion/03-seguridad.md)
- [Sistema de Backup](documentacion/04-backup.md)
- [Monitorización](documentacion/05-monitorizacion.md)

---

## 👤 Autor

**Viktor Paulauskis**

- Proyecto Final ASIR
- **CEAC**
- Año: 2025-2026

---

## 📄 Licencia

Este proyecto está bajo la Licencia MIT - ver el archivo [LICENSE](LICENSE) para más detalles.

---

## 🙏 Agradecimientos

- Profesores del ciclo ASIR
- Comunidad de software libre
- Documentación oficial de Samba, pfSense, Zabbix y demás proyectos utilizados

---

<p align="center">
  <i>Proyecto desarrollado como parte del ciclo formativo ASIR</i>
</p>


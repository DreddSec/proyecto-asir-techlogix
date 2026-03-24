<div align="center">
<img width="435" height="478" alt="lgo_tech" src="https://github.com/user-attachments/assets/fedf8b10-88ab-4566-9a8a-9c2fad9e5539" alt="TechLogix" width="200"/>
<p><strong>TechLogix - Infraestructura de Red Empresarial</strong></p>
</div>

# 🏢 TechLogix - Infraestructura de Red Empresarial

<p align="center">
  <img src="https://img.shields.io/badge/Plataforma-Ubuntu%20Server%2024.04-orange" alt="Plataforma">
  <img src="https://img.shields.io/badge/Virtualizaci%C3%B3n-VirtualBox-blue" alt="Virtualización">
  <img src="https://img.shields.io/badge/Firewall-pfSense-red" alt="Firewall">
  <img src="https://img.shields.io/badge/IDS-Snort-yellow" alt="Snort">
  <img src="https://img.shields.io/badge/Proyecto-ASIR-purple" alt="ASIR">
</p>

---

## 📋 Descripción del Proyecto

> La empresa **TechLogix**, dedicada a la gestión logística, ha decidido modernizar su infraestructura de red para mejorar la eficiencia operativa y garantizar la seguridad de sus datos. Este proyecto se centra en el diseño e implementación de una red informática que soporte sus operaciones, incluyendo la configuración de servidores (On-Premise), dispositivos de red, y la implementación de políticas de seguridad.

---

## 🎯 Objetivos del Proyecto

- Diseñar una arquitectura de red robusta que soporte las operaciones de la empresa
- Implementar un sistema de gestión de datos que permita un acceso rápido y seguro a la información
- Establecer políticas de seguridad para proteger los datos sensibles y garantizar la continuidad operativa
- Optimizar el rendimiento de la red mediante el uso adecuado de tecnologías y herramientas

---

## 📐 Alcance del Proyecto

- Diseño y configuración de la red LAN
- Implementación de un servidor central para gestión de datos
- Configuración de dispositivos de seguridad (firewalls, VPN)
- Establecimiento de un sistema de respaldo y recuperación ante desastres

---

## 🏗️ Arquitectura de Red

### Diagrama de Topología

```
                                          ┌─────────────────┐
                                          │     INTERNET    │
                                          └────────┬────────┘
                                                   │
                                          ┌────────┴────────┐
                                          │    pfSense      │
                                          │   (Firewall)    │
                                          │   Snort IDS     │
                                          └────────┬────────┘
                                                   │
       ┌───────────────────────────────────────────┼─────────────────────|────────────────────┐
       │                    │                      │                     |                    |
┌──────┴──────┐     ┌───────┴───────┐     ┌───────┴───────┐     ┌────────┴───────┐    ┌───────┴───────┐
│  VLAN 40    │     │   VLAN 100    │     │   VLAN 60     │     │    VLAN 70     │    │   VLAN 50     │
│  SERVERS    │     │     DMZ       │     │   SECURITY    │     │   MONITORING   │    │  WIFI GUESTS  │
│192.168.40.0 │     │ 192.168.100.0 │     │  192.168.60.0 │     │  192.168.70.0  │    │  192.168.50.0 │
└──────┬──────┘     └───────┬───────┘     └───────┬───────┘     └────────┬───────┘    └───────────────┘
       │                    │                     │                      │
  ┌────┴────┐          ┌────┴────┐           ┌────┴────┐           ┌─────┴─────┐
  │ DC01    │          │ WEB01   │           │ SEC01   │           │  MON01    │
  │ DC02    │          │WordPress│           │ OpenVPN │           │  Zabbix   │
  │ FILE01  │          └─────────┘           └─────────┘           │  Grafana  │
  │ BAK01   │                                                      │  Ansible  │
  └─────────┘                                                      └───────────┘

       ┌───────────────────────────────────────────────────────────────────┐
       │                              │                                    │
┌──────┴──────┐              ┌────────┴───────┐                   ┌────────┴───────┐
│  VLAN 10    │              │    VLAN 20     │                   │    VLAN 30     │
│   ADMIN     │              │   PRODUCCION   │                   │      IT        │
│192.168.10.0 │              │ 192.168.20.0   │                   │ 192.168.30.0   │
└──────┬──────┘              └────────┬───────┘                   └────────┬───────┘
       │                              │                                    │
  ┌────┴────┐                    ┌────┴────┐                          ┌────┴────┐
  │ WIN-ADM │                    │ Equipos │                          │ CLI-IT  │
  │Windows11│                    │  Prod   │                          │ Debian  │
  └─────────┘                    └─────────┘                          └─────────┘
```

### Segmentación de VLANs

| VLAN ID | Nombre | Subred | Propósito |
|---------|--------|--------|-----------|
| 10 | ADMIN | 192.168.10.0/24 | Administradores de sistemas |
| 20 | PROD | 192.168.20.0/24 | Departamento de producción |
| 30 | IT | 192.168.30.0/24 | Departamento técnico |
| 40 | SERVERS | 192.168.40.0/24 | Servidores internos |
| 50 | WIFI_GUESTS | 192.168.50.0/24 | Red WiFi invitados (aislada) |
| 60 | SECURITY | 192.168.60.0/24 | VPN y servicios de seguridad |
| 70 | MONITORING | 192.168.70.0/24 | Monitorización centralizada |
| 100 | DMZ | 192.168.100.0/24 | Web (Zona Desmilitarizada) |

---

## 🖥️ Inventario de Servidores

| Servidor | IP | Sistema Operativo | Servicios |
|----------|-----|-------------------|-----------|
| SRV-DC01 | 192.168.40.10 | Ubuntu Server 24.04 | Samba AD, DNS, DHCP |
| SRV-DC02 | 192.168.40.14 | Ubuntu Server 24.04 | Samba AD (réplica), DNS |
| SRV-FILE01 | 192.168.40.12 | Ubuntu Server 24.04 | SMB, FTP (vsftpd) |
| SRV-BAK01 | 192.168.40.13 | Ubuntu Server 24.04 | Bacula, RAID 5, Rclone |
| SRV-WEB01 | 192.168.100.8 | Ubuntu Server 24.04 | Apache, WordPress, MySQL |
| SRV-SEC01 | 192.168.60.10 | Ubuntu Server 24.04 | OpenVPN |
| SRV-MON01 | 192.168.70.10 | Ubuntu Server 24.04 | Zabbix, Grafana, Ansible |
| FW-PFSENSE01 | 10.10.10.4 (WAN) | FreeBSD (pfSense) | Firewall, Router, Snort IDS |

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
| Recurso | Ruta | Grupo con acceso |
|---------|------|------------------|
| Comun | /srv/compartido/comun | Domain Users |
| Administracion | /srv/compartido/administracion | GRP_Administracion |
| IT | /srv/compartido/it | GRP_IT |
| Produccion | /srv/compartido/produccion | GRP_Produccion |

**Servidor FTP (vsftpd):**
- Transferencias externas seguras
- Integrado con acceso VPN para trabajo remoto
- Acceso restringido al departamento IT

### 🌐 VPN (OpenVPN)

- **Puerto:** 1194/UDP
- **Autenticación:** Certificados X.509 + validación LDAP contra Samba AD
- **Restricción:** Solo miembros del grupo GRP_IT pueden conectar
- **Cifrado:** AES-256-GCM
- **Acceso remoto:** Usuarios VPN acceden a recursos internos (FTP) desde Internet

### 💾 Sistema de Backup - Estrategia 3-2-1

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   ORIGINAL  │         │   LOCAL     │         │    CLOUD    │
│   (Datos)   │ ──────► │  (RAID 5)   │ ──────► │  (GDrive)   │
│             │         │  SRV-BAK01  │         │   Rclone    │
└─────────────┘         └─────────────┘         └─────────────┘
      1                       2                       3
   Copia                   Copias                 Ubicación
  Original                Locales                  Remota
```

- **Bacula Director:** Orquesta backups de todos los servidores
- **RAID 5:** 5 discos para redundancia local
- **Rclone:** Sincronización automática diaria a Google Drive (cron 3:00 AM)
- **Programación:**
  - Full: Primer domingo del mes
  - Diferencial: Domingos 2-5
  - Incremental: Lunes a Sábado

### 📊 Monitorización

- **Zabbix Server:** Agentes instalados en todos los servidores
- **Métricas:** CPU, RAM, Disco, Red, Servicios
- **Grafana:** Dashboards personalizados con visualización avanzada
- **Alertas:** Notificaciones por email configuradas

---

## 🛡️ Seguridad Implementada

### Firewall Perimetral (pfSense)

- Reglas granulares por VLAN aplicando principio de mínimo privilegio
- Port forwarding selectivo para servicios publicados (OpenVPN)
- NAT para acceso a Internet desde redes internas
- Aislamiento total de WIFI_GUESTS (solo acceso a Internet)

### Sistema de Detección de Intrusiones (Snort)

- Instalado en pfSense monitorizando interfaces: WAN, LAN, DMZ, VPN
- Reglas VRT + Community Rules activadas
- Protección contra escaneos de puertos y ataques DDoS

### Hardening de Servidores (Automatizado con Ansible)

**SSH Seguro:**
- Puerto personalizado (2222)
- Root login deshabilitado
- Autenticación por contraseña deshabilitada (solo claves SSH)
- MaxAuthTries: 3
- X11Forwarding deshabilitado

**Fail2ban:**
- Protección contra fuerza bruta
- Bantime: 3600 segundos
- MaxRetry: 3 intentos

**Auditoría del Sistema (auditd):**
- Monitorización de cambios en `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`
- Registro de conexiones de red
- Logs de autenticación

**Políticas de Contraseñas (PAM):**
- Longitud mínima: 12 caracteres
- Mínimo 3 clases de caracteres
- Requiere mayúsculas, números y caracteres especiales

**Actualizaciones Automáticas:**
- Unattended-upgrades configurado para parches de seguridad
- Limpieza automática de kernels antiguos

**Permisos de Archivos Críticos:**
- `/etc/shadow`: 600
- `/etc/gshadow`: 600
- `/etc/passwd`: 644

**Otras Medidas:**
- NTP sincronizado en todos los servidores
- Límites de recursos (ulimits) configurados
- Banner legal de advertencia en acceso SSH
- Servicios innecesarios deshabilitados (avahi-daemon, cups)

### Firewall Host-based (UFW)

Configurado en cada servidor con reglas específicas por rol:
- Controladores de dominio: Puertos AD/Kerberos (53, 88, 389, 445, 464, 636)
- Servidor de archivos: SMB (445) + FTP (21)
- Servidor VPN: OpenVPN (1194) + LDAP hacia DC
- Todos: Zabbix Agent (10050) + Bacula FD (9102)

---

## 🤖 Automatización con Ansible

Desde **SRV-MON01** se gestionan todos los servidores mediante playbooks:
 

- **securizacion.yml:** Hardening completo de servidores
- **bacula_rclone.yml:** Configuración de sistema de backup

 Autenticación mediante claves SSH desde usuario `ansible` hacia `sysadmin` en todos los servidores.


---

## 🚀 Tecnologías Utilizadas

<p align="center">
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/pfSense-003366?style=for-the-badge&logo=pfsense&logoColor=white" alt="pfSense">
  <img src="https://img.shields.io/badge/Samba-FCC624?style=for-the-badge&logo=samba&logoColor=black" alt="Samba">
  <img src="https://img.shields.io/badge/OpenVPN-EA7E20?style=for-the-badge&logo=openvpn&logoColor=white" alt="OpenVPN">
  <img src="https://img.shields.io/badge/Snort-F80000?style=for-the-badge&logo=snort&logoColor=white" alt="Snort">
  <img src="https://img.shields.io/badge/Zabbix-D50000?style=for-the-badge&logo=zabbix&logoColor=white" alt="Zabbix">
  <img src="https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white" alt="Grafana">
  <img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white" alt="Ansible">
  <img src="https://img.shields.io/badge/VirtualBox-183A61?style=for-the-badge&logo=virtualbox&logoColor=white" alt="VirtualBox">
</p>

---

## 📖 Documentación Adicional

- [Arquitectura de Red](documentacion/01-arquitectura.md)
- [Servicios de Red](documentacion/02-servicios.md)
- [Seguridad](documentacion/03-seguridad.md)
- [Sistema de Backup](documentacion/04-backup.md)
- [Monitorización](documentacion/05-monitorizacion.md)

---

## 👤 Autor

> **Viktor Paulauskis**
- Ciclo Formativo: *ASIR* (Administración de Sistemas Informáticos en Red)
- Curso impartido por: **CEAC**, respaldado por la *Asociación de Técnicos de Informática* (ATI) y el *Instituto Nebrija*.
---

## Conclusión final del proyecto:
> Durante este ciclo formativo de ASIR, he comprendido que la administración de sistemas informáticos en red no se reduce a la configuración de herramientas, sino a la construcción de infraestructuras confiables que equilibren seguridad, disponibilidad y eficiencia operacional. Este proyecto ha evidenciado una verdad fundamental: la seguridad no es un componente aislado, sino un principio que debe permear cada capa de la infraestructura, desde la segmentación de redes hasta la protección de datos y la continuidad del servicio.

> He aprendido que la disponibilidad de los sistemas es un compromiso con los usuarios y el negocio, y que la redundancia, la automatización y el monitoreo constante son pilares indispensables para cumplir con ese compromiso. Asimismo, reconozco que la resiliencia de una infraestructura depende más de decisiones arquitectónicas conscientes que de la cantidad de recursos invertidos. Un administrador de sistemas debe pensar como un gestor de riesgos, anticipando fallos y diseñando soluciones que minimicen el impacto de lo inevitable.

## 📄 Licencia

Este proyecto está bajo la Licencia MIT - ver el archivo [LICENSE](LICENSE) para más detalles.

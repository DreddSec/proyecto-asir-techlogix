# 02 - Servicios de Red

## 2.1 Introducción

Este documento describe los servicios de red implementados en la infraestructura TechLogix, incluyendo Active Directory, DNS, DHCP, servidor de archivos (SMB/FTP) y servidor web.

---

## 2.2 Active Directory (Samba AD)

### 2.2.1 Descripción General

Se ha implementado un dominio Active Directory utilizando **Samba 4** como alternativa open source a Windows Server. Esto proporciona:

- Autenticación centralizada
- Políticas de grupo (GPO)
- DNS integrado
- Servicios Kerberos

### 2.2.2 Configuración del Dominio

| Parámetro | Valor |
|-----------|-------|
| Nombre de Dominio | TECHLOGIX.LOCAL |
| Controlador Primario | SRV-DC01 (192.168.40.10) |
| Controlador Secundario | SRV-DC02 (192.168.40.14) |

### 2.2.3 Estructura Organizativa (OUs)

```
TECHLOGIX.LOCAL
│
├── OU=Usuarios
│   ├── OU=Administracion
│   │   └── carlos.ruiz
│   │   └── maria.lopez
│   │
│   ├── OU=IT
│   │   └── david.garcia
│   │   └── ana.martinez
│   │
│   └── OU=Produccion
│       └── pedro.sanchez
│       └── laura.fernandez
│
├── OU=Grupos
    ├── GRP_Administracion
    ├── GRP_IT
    └── GRP_Produccion


```

### 2.2.4 Usuarios del Sistema

| Usuario | Nombre Completo | Departamento | Grupo |
|---------|-----------------|--------------|-------|
| carlos.ruiz | Carlos Ruiz | Administración | GRP_Administracion |
| maria.lopez | María López | Administración | GRP_Administracion |
| david.garcia | David García | IT | GRP_IT |
| ana.martinez | Ana Martínez | IT | GRP_IT |
| pedro.sanchez | Pedro Sánchez | Producción | GRP_Produccion |
| laura.fernandez | Laura Fernández | Producción | GRP_Produccion |


### 2.2.6 Archivos de Configuración

**Ubicación: DC01** `/etc/samba/smb.conf`

```ini
[global]
    workgroup = TECHLOGIX
    realm = TECHLOGIX.LOCAL
    netbios name = SRV-DC01
    server role = active directory domain controller
    dns forwarder = 192.168.40.1
    bind interfaces only = yes
    interfaces = lo 192.168.40.10/24
    ldap server require strong auth = no
    encrypt password = yes
    
    # Permitir autenticación LDAP sin cifrar (para OpenVPN)
    ldap server require strong auth = no

[sysvol]
    path = /var/lib/samba/sysvol
    read only = No

[netlogon]
    path = /var/lib/samba/sysvol/techlogix.local/scripts
    read only = No
```

---

## 2.3 DNS

### 2.3.1 Arquitectura DNS

```
                    ┌─────────────────┐
                    │   pfSense       │
                    │ DNS Resolver    │
                    │ (Unbound)       │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
    ┌─────────────────┐           ┌─────────────────┐
    │    Consultas    │           │    Consultas    │
    │    Externas     │           │ techlogix.local │
    │   (google.com)  │           │                 │
    └────────┬────────┘           └────────┬────────┘
             │                             │
             ▼                             ▼
    ┌─────────────────┐           ┌─────────────────┐
    │   DNS Público   │           │   DC01 / DC02   │
    │   (8.8.8.8)     │           │   Samba DNS     │
    └─────────────────┘           └─────────────────┘
```

### 2.3.2 Domain Override en pfSense

Para que las consultas del dominio interno se resuelvan correctamente:

**Services → DNS Resolver → Domain Overrides:**

| Domain | IP Address | Description |
|--------|------------|-------------|
| techlogix.local | 192.168.40.10 | Forward to DC01 |
| techlogix.local | 192.168.40.14 | Forward to DC02 |

### 2.3.3 Registros DNS Internos

| Registro | Tipo | Valor |
|----------|------|-------|
| srv-dc01.techlogix.local | A | 192.168.40.10 |
| srv-dc02.techlogix.local | A | 192.168.40.14 |
| srv-file01.techlogix.local | A | 192.168.40.12 |
| srv-bak01.techlogix.local | A | 192.168.40.13 |
| srv-web01.techlogix.local | A | 192.168.100.8 |
| srv-sec01.techlogix.local | A | 192.168.60.10 |
| srv-mon01.techlogix.local | A | 192.168.70.10 |

---

## 2.4 DHCP

### 2.4.1 Configuración por VLAN

El servicio DHCP se proporciona desde DC01/02 para cada VLAN de clientes:

| VLAN | Rango DHCP | Gateway | DNS |
|------|------------|---------|-----|
| ADMIN (10) | 192.168.10.100 - .200 | 192.168.10.1 | 192.168.40.10 |
| PROD (20) | 192.168.20.100 - .200 | 192.168.20.1 | 192.168.40.10 |
| IT (30) | 192.168.30.100 - .200 | 192.168.30.1 | 192.168.40.10 |
| WIFI_GUESTS (50) | 192.168.50.10 - .200 | 192.168.50.1 | 8.8.8.8 |

 **WIFI_GUESTS** usa *DNS público* (8.8.8.8) para evitar acceso a resolución de nombres internos.


## 2.5 Servidor de Archivos (SMB)

### 2.5.1 Descripción

SRV-FILE01 actúa como servidor de archivos utilizando Samba integrado con el dominio Active Directory para autenticación y permisos basados en grupos.

### 2.5.2 Estructura de Carpetas

```
/srv/compartido/
├── comun/              # Acceso: Todos los usuarios del dominio
├── administracion/     # Acceso: GRP_Administracion
├── it/                 # Acceso: GRP_IT
└── produccion/         # Acceso: GRP_Produccion
```

### 2.5.3 Configuración SMB

**Archivo:** `/etc/samba/smb.conf`

```ini
[global]
    workgroup = TECHLOGIX
    realm = TECHLOGIX.LOCAL
    security = ADS
    log level = 3 auth:5
    log file = /var/log/samba/log.%m
    max log size = 50000
    
    # Winbind
    winbind use default domain = yes
    winbind enum users = yes
    winbind enum groups = yes
    winbind separator = +

    # Mapeo de IDs
    idmap config * : backend = tdb
    idmap config * : range = 3000-7999
    idmap config TECHLOGIX : backend = rid
    idmap config TECHLOGIX : range = 10000-999999
    template shell = /bin/bash

[Comun]
    path = /srv/compartido/comun
    browseable = yes
    read only = no
    valid users = @"TECHLOGIX\Domain Users"
    create mask = 0664
    directory mask = 0775

[Administracion]
    path = /srv/compartido/administracion
    browseable = yes
    read only = no
    valid users = @"TECHLOGIX+GRP_Administracion"
    create mask = 0660
    directory mask = 0770
    audit = yes

[IT]
    path = /srv/compartido/it
    browseable = yes
    read only = no
    valid users = @"TECHLOGIX+GRP_IT"
    create mask = 0660
    directory mask = 0770
    audit = yes

[Produccion]
    path = /srv/compartido/produccion
    browseable = yes
    read only = no
    valid users = @"TECHLOGIX+GRP_Produccion"
    create mask = 0660
    directory mask = 0770
    audit = yes
```

### 2.5.4 Permisos de Sistema de Archivos

```bash
# Propietarios y grupos
root:grp_administracion /srv/compartido/administracion
root:grp_it /srv/compartido/it
root:grp_produccion /srv/compartido/produccion
root:domain users /srv/compartido/comun

# Permisos
drwxrws--- /srv/compartido/administracion
drwxrws--- /srv/compartido/it
drwxrws--- /srv/compartido/produccion
drwxrwsr-x /srv/compartido/comun
```

### 2.5.5 Acceso desde Clientes

**Windows:**
```
\\192.168.40.12\Administracion
\\192.168.40.12\IT
\\srv-file01.techlogix.local\Comun
```

**Linux:**
```bash
smbclient //192.168.40.12/IT -U david.garcia
mount -t cifs //192.168.40.12/IT /mnt/it -o user=david.garcia
```

---

## 2.6 Servidor FTP

### 2.6.1 Descripción

Configurado en SRV-FILE01 *vsftpd* para transferencias de archivos seguras, principalmente utilizado por personal IT mediante acceso VPN.

### 2.6.2 Configuración

**Archivo:** `/etc/vsftpd.conf`

```ini
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
allow_writeable_chroot=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=10100
```

### 2.6.4 Acceso via VPN

Los usuarios IT conectan mediante *VPN* y acceden al servidor FTP:

```bash
# Conexion conarchivo .opvn del cliente
sudo openvpn --config david-vpn.ovpn

# Desde cliente VPN (IP 10.8.0.x)
ftp 192.168.40.12
```

---

## 2.7 Servidor Web (LAMP + WordPress)

### 2.7.1 Stack Tecnológico

| Componente | Versión | Función |
|------------|---------|---------|
| Ubuntu Server | 24.04 LTS | Sistema Operativo |
| Apache | 2.4.x | Servidor Web |
| MySQL | 8.0.x | Base de Datos |
| PHP | 8.3.x | Lenguaje Backend |
| WordPress | Latest | CMS |

### 2.7.2 Configuración Apache

**Virtual Host:** `/etc/apache2/sites-available/wordpress.conf`

```apache
<VirtualHost *:80>
    ServerAdmin admin@techlogix.local
    DocumentRoot /var/www/wordpress
    ServerName srv-web01.techlogix.local
    
    <Directory /var/www/wordpress>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog ${APACHE_LOG_DIR}/wordpress_error.log
    CustomLog ${APACHE_LOG_DIR}/wordpress_access.log combined
</VirtualHost>
```

### 2.7.3 Base de Datos

```sql
CREATE DATABASE wordpress;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
```

### 2.7.4 Acceso

- **Interno:** http://192.168.100.8
- **Dominio:** http://srv-web01.techlogix.local (desde red interna)

---

## 2.8 NTP (Sincronización Horaria)

### 2.8.1 Importancia

La sincronización horaria es crítica para:
- Autenticación Kerberos (tolerancia de 5 minutos)
- Correlación de logs
- Certificados y timestamps

### 2.8.2 Configuración

Todos los servidores sincronizan contra pfSense, que a su vez sincroniza contra servidores NTP públicos.

```bash
# Verificar sincronización
timedatectl status
ntpq -p
```

---

## 2.9 Resumen de Puertos por Servicio

| Servicio | Puerto | Protocolo | Servidor |
|----------|--------|-----------|----------|
| DNS | 53 | TCP/UDP | DC01, DC02 |
| Kerberos | 88 | TCP/UDP | DC01, DC02 |
| LDAP | 389 | TCP | DC01, DC02 |
| LDAPS | 636 | TCP | DC01, DC02 |
| SMB | 445 | TCP | DC01, DC02, FILE01 |
| NetBIOS | 139 | TCP | DC01, DC02, FILE01 |
| FTP | 21 | TCP | FILE01 |
| FTP Pasivo | 40000-50000 | TCP | FILE01 |
| HTTP | 80 | TCP | WEB01, MON01 |
| HTTPS | 443 | TCP | WEB01 |
| MySQL | 3306 | TCP | WEB01 |
| SSH | 2222 | TCP | Todos |

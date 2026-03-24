# 02 - Servicios de Red

## 2.1 Introducción

> Este documento describe los servicios de red implementados en la infraestructura TechLogix, incluyendo **Active Directory**, **DNS**, **DHCP**, servidor de archivos SMB/FTP y servidor web.

---

## 2.2 Active Directory (Samba AD)

### 2.2.1 Descripción General

> Se ha implementado un dominio Active Directory utilizando **Samba 4** como alternativa open source a Windows Server. Esto proporciona:

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
<img width="488" height="173" alt="domain_info" src="https://github.com/user-attachments/assets/2269eb28-b291-40cb-a1a4-665f4b9887b3" />

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
│   │   └── laura.fernandez
│   │
│   └── OU=Produccion
│       └── pedro.sanchez
│       └── jorge.navarro
│
├── OU=Grupos
    ├── GRP_Administracion
    ├── GRP_IT
    └── GRP_Produccion

```
<img width="366" height="153" alt="ou_list" src="https://github.com/user-attachments/assets/c92b7297-5f0b-4de3-ad6e-e55733c2a496" />


### 2.2.4 Usuarios del Sistema

| Usuario | Nombre Completo | Departamento | Grupo |
|---------|-----------------|--------------|-------|
| carlos.ruiz | Carlos Ruiz | Administración | GRP_Administracion |
| maria.lopez | María López | Administración | GRP_Administracion |
| david.garcia | David García | IT | GRP_IT |
| laura.fernandez | Laura Fernández | IT | GRP_IT |
| pedro.sanchez | Pedro Sánchez | Producción | GRP_Produccion |
| jorge.navarro | Jorge Navarro | Producción | GRP_Produccion |
<img width="381" height="216" alt="user_list" src="https://github.com/user-attachments/assets/0bf71670-011b-4452-bb4f-f16cdf43e382" />


### 2.2.5 Archivos de Configuración

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
<img width="805" height="592" alt="dns_query" src="https://github.com/user-attachments/assets/b5e12e46-4a7b-4255-9156-0c531477bb17" />

---

## 2.4 DHCP

### 2.4.1 Configuración por VLAN

El servicio DHCP se proporciona desde **pfSense** para cada VLAN de clientes:

| VLAN | Rango DHCP | Gateway | DNS |
|------|------------|---------|-----|
| ADMIN (10) | 192.168.10.100 - .200 | 192.168.10.1 | 192.168.40.10 |
| PROD (20) | 192.168.20.100 - .200 | 192.168.20.1 | 192.168.40.10 |
| IT (30) | 192.168.30.100 - .200 | 192.168.30.1 | 192.168.40.10 |
| WIFI_GUESTS (50) | 192.168.50.10 - .200 | 192.168.50.1 | 8.8.8.8 (Goolge) |
<img width="765" height="743" alt="image" src="https://github.com/user-attachments/assets/ed8fdac4-b152-4bc1-8859-d41dc2ecdd46" />

<img width="761" height="719" alt="dhcp-server-opt7" src="https://github.com/user-attachments/assets/54a00925-594a-44e1-9c10-4eb98b499ff2" />
 
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

### 2.5.2 Permisos de Sistema de Archivos

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
<img width="575" height="151" alt="carpetas_permisos_srv" src="https://github.com/user-attachments/assets/349b3471-57af-46d7-a5fd-051db2365cba" />


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
    valid users = @"TECHLOGIX+Domain Users"
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

### 2.5.4 Acceso desde Clientes

**Windows:**
```
\\192.168.40.12\Administracion
\\192.168.40.12\IT
\\srv-file01.techlogix.local\Comun
```
<img width="1132" height="638" alt="win_smb" src="https://github.com/user-attachments/assets/dc886afd-7f89-4525-9385-6337b7257053" />
<img width="1219" height="638" alt="carlos_smb_admin" src="https://github.com/user-attachments/assets/8eba804d-8269-4663-86ba-65d3a91d91ff" />

- **Carlos Ruiz** inicio sesion en su workstation y pudo acceder sin problemas a la carpeta administracion debido a sus permisos.


<img width="885" height="727" alt="acceso_deny_smb" src="https://github.com/user-attachments/assets/b84a284c-977a-4628-ba20-6c2d1761a9e2" />

- Intentando acceder a una carpeta sin los permisos necesarios.

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

### 2.6.3 Acceso via VPN

Los usuarios IT conectan mediante *VPN* y acceden al servidor FTP:

```bash
# Conexion conarchivo .opvn del cliente
sudo openvpn --config david-vpn.ovpn
```
<img width="1781" height="591" alt="david_vpn_conn" src="https://github.com/user-attachments/assets/c516eb7b-b482-4fd9-92a2-71aad6f2fc6b" />

```bash
# Desde cliente VPN como David Garcia (IP 10.8.0.x)
smbclient //192.168.40.12/it
```
<img width="765" height="208" alt="smblcient_proof" src="https://github.com/user-attachments/assets/6334072a-e2a0-46d9-a510-2f60c7a68abb" />


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
<img width="1660" height="901" alt="home_page_techlogix" src="https://github.com/user-attachments/assets/9999ca8d-9b56-4093-bd04-c7f737f454cc" />
<img width="1261" height="925" alt="dashboard_WP" src="https://github.com/user-attachments/assets/f96bfee8-6330-4c2e-8400-8f72a33192a2" />


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
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY '[PASSWORD]';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
```
<img width="636" height="704" alt="mysql" src="https://github.com/user-attachments/assets/7f6fcaf7-8c7f-4d20-919a-4d9cdb8dbcbc" />


### 2.7.4 Acceso

- **Interno:** http://192.168.100.8
- **Dominio:** http://srv-web01.techlogix.local (desde red interna)
<img width="599" height="138" alt="curl_web" src="https://github.com/user-attachments/assets/656efca0-2c7e-4293-abcf-d9726a2eca12" />
<img width="1337" height="467" alt="web_dns" src="https://github.com/user-attachments/assets/a5a806cf-ee57-4d64-b940-08792af51157" />


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
<img width="842" height="316" alt="ntp" src="https://github.com/user-attachments/assets/80afa799-11b8-4cad-b80b-54a9922117b1" />


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

# 03 - Seguridad

## 3.1 Introducción

> Este documento describe las medidas de seguridad implementadas en la infraestructura TechLogix siguiendo el principio de **defensa en profundidad** con múltiples capas de protección.

---

## 3.2 Arquitectura de Seguridad

### 3.2.1 Modelo de Defensa en Profundidad

```
┌─────────────────────────────────────────────────────────────────┐
│                        CAPA 1: PERÍMETRO                        │
│                     pfSense Firewall + NAT                      │
│                            Snort IDS                            │
└─────────────────────────────────────────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                      CAPA 2: SEGMENTACIÓN                       │
│                    VLANs + Reglas Inter-VLAN                    │
│                     Aislamiento de Zonas                        │
└─────────────────────────────────────────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                        CAPA 3: HOST                             │
│              UFW (Firewall) + Fail2ban + Auditd                 │
│                     Hardening SSH + PAM                         │
└─────────────────────────────────────────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                     CAPA 4: APLICACIÓN                          │
│           Autenticación AD + Permisos Granulares                │
│                    Cifrado (TLS/SSL)                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Firewall Perimetral (pfSense)

### 3.3.1 Políticas Generales

- **Política por defecto:** Denegar todo (deny all)
- **Principio:** Mínimo privilegio - solo permitir lo necesario
- **Logging:** Habilitado para reglas de bloqueo

### 3.3.2 Reglas por Interfaz

#### WAN (Internet)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | UDP | * | 192.168.60.10 | 1194 | OpenVPN NAT Port Forward |
<img width="1025" height="210" alt="WAN_RULESFW" src="https://github.com/user-attachments/assets/7699510d-402f-484e-86b5-b373f2cadca3" />
<img width="1023" height="212" alt="port_forward_vpn" src="https://github.com/user-attachments/assets/7ccccf25-a278-4ef9-885c-ad9290fb519b" />


---
#### LAN (Servers - 192.168.40.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | * | * | LAN Address | 443, 80, 22 | Anti-Lockout pfSense |
| Pass | TCP/UDP | LAN subnets | * | 123 | NTP |
| Pass | TCP | LAN subnets | OPT3 subnets | 10051 | Zabbix active checks (servidores → MON01) |
| Pass | UDP | LAN subnets | * | 53 | DNS |
| Pass | TCP | LAN subnets | * | 80, 443 | HTTP/HTTPS (actualizaciones) |
| Pass | TCP | 192.168.40.13 | OPT1 subnets | 9102 | Bacula Director → FD WEB01 |
| Pass | TCP | 192.168.40.13 | OPT2 subnets | 9102 | Bacula Director → FD SEC01 |
| Pass | TCP | 192.168.40.13 | OPT3 subnets | 9102 | Bacula Director → FD MON01 |
<img width="1022" height="506" alt="LAN_RULES" src="https://github.com/user-attachments/assets/5edfdc0d-1172-4335-a791-47b1d9942c00" />

---
#### OPT1 (DMZ - 192.168.100.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Block | * | OPT1 subnets | LAN subnets | * | Block → Servers |
| Block | * | OPT1 subnets | OPT2 subnets | * | Block → VPN/SEC |
| Block | * | OPT1 subnets | OPT3 subnets | * | Block → MON |
| Block | * | OPT1 subnets | OPT4 subnets | * | Block → ADMIN |
| Block | * | OPT1 subnets | OPT5 subnets | * | Block → PROD |
| Block | * | OPT1 subnets | OPT6 subnets | * | Block → IT |
| Pass | TCP | OPT1 subnets | 192.168.40.13 | 9103 | Bacula FD → SD (WEB01 → BAK01) |
| Pass | TCP | OPT1 subnets | OPT3 subnets | 10051 | Zabbix active checks (WEB01 → MON01) |
| Pass | UDP | OPT1 subnets | * | 53 | DNS |
| Pass | TCP | OPT1 subnets | * | 80 | HTTP |
| Pass | TCP | OPT1 subnets | * | 443 | HTTPS |
<img width="1025" height="537" alt="OPT1_RULES" src="https://github.com/user-attachments/assets/90822d0b-5acd-4d78-9f30-80d39cca37a1" />


---
#### OPT2 (Security/VPN - 192.168.60.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | TCP/UDP | 10.8.0.0/24 | 192.168.40.10 | 88 | VPN to Kerberos |
| Pass | TCP/UDP | OPT2 subnets | 192.168.40.10 | 389 | LDAP → DC01 |
| Pass | TCP/UDP | 10.8.0.0/24 | 192.168.40.10 | 389 | VPN to DC |
| Pass | TCP/UDP | OPT2 subnets | 192.168.40.10 | 636 | LDAPS → DC01 |
| Pass | TCP/UDP | 10.8.0.0/24 | 192.168.40.10 | 636 | VPN to DC |
| Pass | TCP/UDP | 10.8.0.0/24 | 192.168.40.10 | 53 | VPN to DNS |
| Pass | TCP | 10.8.0.0/24 | 192.168.40.12 | 445 | VPN to SMB |
| Pass | TCP | 10.8.0.0/24 | 192.168.40.12 | 21 | VPN to FTP |
| Pass | TCP | OPT2 subnets | OPT3 subnets | 10051 | Zabbix active checks (SEC01 → MON01) |
| Pass | TCP | OPT2 subnets | 192.168.40.13 | 9103 | Bacula FD → SD (SEC01 → BAK01) |
| Pass | UDP | OPT2 subnets | * | 53 | DNS general |
| Pass | TCP | OPT2 subnets | * | 80 | HTTP |
| Pass | TCP | OPT2 subnets | * | 443 | HTTPS |
<img width="1027" height="598" alt="OPT2_RULES" src="https://github.com/user-attachments/assets/3ca70679-9f19-4908-b6c9-39755eab550c" />


---
#### OPT3 (Monitoring - 192.168.70.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | TCP | OPT3 subnets | LAN subnets | 10050 | Zabbix polling → Servers |
| Pass | TCP | OPT3 subnets | OPT1 subnets | 10050 | Zabbix polling → WEB01 |
| Pass | TCP | OPT3 subnets | OPT2 subnets | 10050 | Zabbix polling → SEC01 |
| Pass | TCP | OPT3 subnets | LAN subnets | 2222 | SSH Ansible → Servers |
| Pass | TCP | OPT3 subnets | OPT1 subnets | 2222 | SSH Ansible → WEB01 |
| Pass | TCP | OPT3 subnets | OPT2 subnets | 2222 | SSH Ansible → SEC01 |
| Pass | TCP | OPT3 subnets | 192.168.40.13 | 9103 | Bacula FD → SD (MON01 → BAK01) |
| Pass | TCP | OPT3 subnets | * | 587 | SMTP alertas |
| Pass | UDP | OPT3 subnets | * | 53 | DNS |
| Pass | TCP | OPT3 subnets | * | 80 | HTTP |
| Pass | TCP | OPT3 subnets | * | 443 | HTTPS |
<img width="1024" height="528" alt="OPT3_RULES" src="https://github.com/user-attachments/assets/1026fff4-d97d-4428-ae63-5e391777b598" />

---
#### OPT4 (Admin - 192.168.10.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Block | * | OPT4 subnets | OPT1 subnets | * | Block → DMZ |
| Pass | TCP/UDP | OPT4 subnets | 192.168.40.10 | 53 | DNS → DC01 |
| Pass | TCP/UDP | OPT4 subnets | 192.168.40.10 | 88 | Kerberos → DC01 |
| Pass | TCP | OPT4 subnets | 192.168.40.10 | 389 | LDAP → DC01 |
| Pass | TCP | OPT4 subnets | 192.168.40.10 | 636 | LDAPS → DC01 |
| Pass | TCP/UDP | OPT4 subnets | 192.168.40.10 | 464 | Kerberos pwd change → DC01 |
| Pass | TCP | OPT4 subnets | 192.168.40.10 | 445 | SMB → DC01 |
| Pass | TCP | OPT4 subnets | 192.168.40.12 | 445 | SMB → FILE01 |
| Pass | TCP | OPT4 subnets | OPT3 subnets | 80 | Zabbix Web → MON01 |
| Pass | TCP | OPT4 subnets | OPT3 subnets | 3000 | Grafana → MON01 |
| Pass | UDP | OPT4 subnets | * | 53 | DNS general |
| Pass | TCP | OPT4 subnets | * | 443 | HTTPS |
<img width="910" height="556" alt="OPT4_RULES" src="https://github.com/user-attachments/assets/b1b9359a-2f99-4d87-8013-23926bd2716e" />

---
#### OPT5 (Producción - 192.168.20.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Block | * | OPT5 subnets | OPT1 subnets | * | Block → DMZ |
| Pass | TCP/UDP | OPT5 subnets | 192.168.40.10 | 53 | DNS → DC01 |
| Pass | TCP/UDP | OPT5 subnets | 192.168.40.10 | 88 | Kerberos → DC01 |
| Pass | TCP | OPT5 subnets | 192.168.40.10 | 389 | LDAP → DC01 |
| Pass | TCP | OPT5 subnets | 192.168.40.10 | 636 | LDAPS → DC01 |
| Pass | TCP/UDP | OPT5 subnets | 192.168.40.10 | 464 | Kerberos pwd change → DC01 |
| Pass | TCP | OPT5 subnets | 192.168.40.10 | 445 | SMB → DC01 |
| Pass | TCP | OPT5 subnets | 192.168.40.12 | 445 | SMB → FILE01 |
| Pass | UDP | OPT5 subnets | * | 53 | DNS general |
| Pass | TCP | OPT5 subnets | * | 80 | HTTP |
| Pass | TCP | OPT5 subnets | * | 443 | HTTPS |
<img width="1025" height="504" alt="OPT5_RULES" src="https://github.com/user-attachments/assets/bf69b2c6-9429-4ae6-b5f9-ca5dfbf0f470" />

---
#### OPT6 (IT — 192.168.30.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Block | * | OPT6 subnets | OPT1 subnets | * | Block → DMZ |
| Pass | TCP/UDP | OPT6 subnets | 192.168.40.10 | 53 | DNS → DC01 |
| Pass | TCP/UDP | OPT6 subnets | 192.168.40.10 | 88 | Kerberos → DC01 |
| Pass | TCP | OPT6 subnets | 192.168.40.10 | 389 | LDAP → DC01 |
| Pass | TCP | OPT6 subnets | 192.168.40.10 | 636 | LDAPS → DC01 |
| Pass | TCP/UDP | OPT6 subnets | 192.168.40.10 | 464 | Kerberos pwd change → DC01 |
| Pass | TCP | OPT6 subnets | 192.168.40.10 | 445 | SMB → DC01 |
| Pass | TCP | OPT6 subnets | 192.168.40.12 | 445 | SMB → FILE01 |
| Pass | TCP | OPT6 subnets | LAN subnets | 2222 | SSH → Servers (gestión IT) |
| Pass | TCP | OPT6 subnets | OPT3 subnets | 80 | Zabbix Web → MON01 |
| Pass | TCP | OPT6 subnets | OPT3 subnets | 3000 | Grafana → MON01 |
| Pass | UDP | OPT6 subnets | * | 53 | DNS general |
| Pass | TCP | OPT6 subnets | * | 80 | HTTP |
| Pass | TCP | OPT6 subnets | * | 443 | HTTPS |
<img width="1022" height="532" alt="OPT6_RULES" src="https://github.com/user-attachments/assets/e40816c2-5409-4317-b99d-e75dfc7bc957" />

----
#### OPT7 (WiFi Guests — 192.168.50.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Block | * | OPT7 subnets | 192.168.0.0/16 | * | Block → red privada |
| Block | * | OPT7 subnets | 10.0.0.0/8 | * | Block → VPN |
| Pass | UDP | OPT7 subnets | * | 53 | DNS internet |
| Pass | TCP | OPT7 subnets | * | 80 | HTTP |
| Pass | TCP | OPT7 subnets | * | 443 | HTTPS |
| Block | * | OPT7 subnets | * | * | Block resto (catch-all) |
<img width="1024" height="373" alt="OPT7_RULES" src="https://github.com/user-attachments/assets/5047aae2-afc5-4c1b-98fe-118ffc35c3d6" />

---

## 3.4 Sistema de Detección de Intrusiones (Snort)

### 3.4.1 Configuración General

- **Ubicación:** Integrado en pfSense
- **Modo:** IDS
- **Interfaces monitorizadas:** WAN, LAN, OPT1 (DMZ), OPT2 (VPN)
<img width="1026" height="316" alt="SNORT" src="https://github.com/user-attachments/assets/62d2aa53-71e2-419b-888a-4670bb068046" />


### 3.4.2 Reglas Activadas (IPS Policy)

> Politicas IPS predefenidas en las siguientes interfaces:

| Interfaz | Politica |
|-----------|-------------|
| WAN | Connectivity |
| LAN | Balancend |
| OPT1 (DMZ) | Security |
| OPT2 (SEC) | Balanced |

### 3.4.3 Preprocesadores

| Preprocesador | Estado | Función |
|---------------|--------|---------|
| HTTP Inspect | ✅ | Análisis tráfico HTTP |
| Portscan Detection | ✅ | Detecta escaneos nmap |
| SSH | ✅ | Ataques SSH |
| FTP/Telnet | ✅ | Protocolos legacy |

### 3.4.4 Configuración Portscan

```
Sensitivity Level: Medium
Detect Scan Type: All
Memory Cap: 10000000
```

## 3.5 VPN (OpenVPN)

### 3.5.1 Características de Seguridad

| Parámetro | Valor |
|-----------|-------|
| Protocolo | UDP |
| Puerto | 1194 |
| Cifrado | AES-256-GCM |
| Autenticación | Certificados X.509 + LDAP |
| Hash | SHA256 |
| TLS Auth | Habilitado |

### 3.5.2 Autenticación LDAP

```ini
# /etc/openvpn/auth-ldap.conf
<LDAP>
    URL             ldap://192.168.40.10
    BindDN          "CN=Administrator,CN=Users,DC=techlogix,DC=local"
    Password        [REDACTED]
    Timeout         15
    TLSEnable       no
</LDAP>

<Authorization>
    BaseDN          "OU=Usuarios,DC=techlogix,DC=local"
    SearchFilter    "(sAMAccountName=%u)"
    RequireGroup    true
    
    <Group>
        BaseDN      "OU=Grupos,DC=techlogix,DC=local"
        SearchFilter "(cn=GRP_IT)"
        MemberAttribute member
    </Group>
</Authorization>
```
> Solo los miembros del grupo **GRP_IT** pueden conectarse via VPN. Otros usuarios del dominio son rechazados.

### 3.5.3 Certificados

| Archivo | Ubicación | Propósito |
|---------|-----------|-----------|
| ca.crt | /etc/openvpn/server/ | Autoridad Certificadora |
| server.crt | /etc/openvpn/server/ | Certificado servidor |
| server.key | /etc/openvpn/server/ | Clave privada servidor |
<img width="486" height="163" alt="crts" src="https://github.com/user-attachments/assets/a2f3d666-e47f-4f64-aac5-ff4882cedb52" />

---

## 3.6 Hardening de Servidores

### 3.6.1 SSH Seguro

**Archivo:** `/etc/ssh/sshd_config`

```bash
# Puerto personalizado
Port 2222

# Deshabilitar root login
PermitRootLogin no

# Solo autenticación por clave
PasswordAuthentication no
PubkeyAuthentication yes

# Limitar intentos
MaxAuthTries 3

# Deshabilitar funciones innecesarias
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no

# Timeout de sesión
ClientAliveInterval 300
ClientAliveCountMax 2
```

### 3.6.2 Fail2ban

**Archivo:** `/etc/fail2ban/jail.local`

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = 2222
logpath = /var/log/auth.log
maxretry = 3
```

**Comandos útiles:**
```bash
# Ver IPs baneadas
fail2ban-client status sshd

# Desbanear IP
fail2ban-client set sshd unbanip 192.168.1.100
```

### 3.6.3 Políticas de Contraseñas (PAM)

**Archivo:** `/etc/security/pwquality.conf`

```ini
# Longitud mínima 12 caracteres
minlen = 12

# Mínimo 3 clases de caracteres
minclass = 3

# Requiere mayúscula
ucredit = -1

# Requiere número
dcredit = -1

# Requiere carácter especial
ocredit = -1
```

### 3.6.4 Auditoría del Sistema (auditd)

**Archivo:** `/etc/audit/rules.d/auth.rules`

```bash
# Monitorizar archivos críticos
-w /etc/passwd -p wa -k passwd_changes
-w /etc/group -p wa -k group_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/gshadow -p wa -k gshadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /var/log/auth.log -p wa -k auth_log_changes
```

**Archivo:** `/etc/audit/rules.d/network.rules`

```bash
# Monitorizar conexiones de red
-a always,exit -F arch=b64 -S socket -S connect -k network_connections
-w /etc/hosts -p wa -k hosts_changes
-w /etc/network/ -p wa -k network_changes
```

**Comandos útiles:**
```bash
# Ver logs de auditoría
ausearch -k passwd_changes
aureport --summary
```

### 3.6.5 Permisos de Archivos Críticos

```bash
chmod 600 /etc/shadow
chmod 600 /etc/gshadow
chmod 644 /etc/passwd
chmod 644 /etc/group
```
<img width="606" height="106" alt="critic_files" src="https://github.com/user-attachments/assets/c126bf99-cdb1-492c-a6f8-fb2cdb13bae4" />


### 3.6.6 Actualizaciones Automáticas

**Archivo:** `/etc/apt/apt.conf.d/50unattended-upgrades`

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";
```

### 3.6.7 Banner Legal

**Archivo:** `/etc/motd`

```
********************************************************************************
ADVERTENCIA: Este sistema es de uso exclusivo para personal autorizado.

El acceso no autorizado está prohibido y séra perseguido legalmente.
Todas las actividades en este sistema son monitorizadas y registradas.

Si no está autorizado, desconéctese inmediatamente.
********************************************************************************
```
<img width="766" height="689" alt="motd" src="https://github.com/user-attachments/assets/99412b76-4bf6-4e36-b197-3ca29290b716" />

---

## 3.7 Firewall Host-based (UFW)

### 3.7.1 Política Base

```bash
ufw default deny incoming
ufw default allow outgoing
```

### 3.7.2 Reglas por Servidor

#### Controladores de Dominio (DC01, DC02)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow 53 comment "DNS"
ufw allow 88 comment "Kerberos"
ufw allow 464 comment "Kerberos Password"
ufw allow 389 comment "LDAP"
ufw allow 636 comment "LDAPS"
ufw allow 3268/tcp comment "Global Catalog"
ufw allow 3269/tcp comment "Global Catalog SSL"
ufw allow 445/tcp comment "SMB"
ufw allow 139/tcp comment "NetBIOS"
ufw allow 137/udp comment "NetBIOS Name"
ufw allow 138/udp comment "NetBIOS Datagram"
ufw allow 135/tcp comment "RPC Endpoint Mapper"
ufw allow from 192.168.40.0/24 to any port 49152:65535 proto tcp comment "RPC Dynamic"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix Agent"
ufw allow from 192.168.40.13 to any port 9101:9104 proto tcp comment "Bacula"
```
<img width="779" height="575" alt="ufw_status" src="https://github.com/user-attachments/assets/f6c9fc12-08dc-483e-8092-a410f263b2ff" />


#### Servidor de Archivos (FILE01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow 21/tcp comment "FTP"
ufw allow 10000:10100/tcp comment "FTP Pasivo"
ufw allow from 192.168.10.0/24 to any port 445 proto tcp comment "SMB Admin"
ufw allow from 192.168.30.0/24 to any port 445 proto tcp comment "SMB IT"
ufw allow from 192.168.20.0/24 to any port 445 proto tcp comment "SMB Produccion"
ufw allow from 10.8.0.0/24 to any port 445 proto tcp comment "SMB VPN"
ufw allow from 10.8.0.0/24 to any port 21 proto tcp comment "FTP VPN"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix Agent"
ufw allow from 192.168.40.13 to any port 9101:9104 proto tcp comment "Bacula"
```
<img width="723" height="304" alt="ufw_status" src="https://github.com/user-attachments/assets/f7ba9480-6d71-4022-8f4d-ce26cfae4a9c" />


#### Servidor VPN (SEC01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow 1194/udp comment "OpenVPN"
ufw allow in on tun0 comment "VPN Traffic"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix Agent"
ufw allow from 192.168.40.13 to any port 9101:9104 proto tcp comment "Bacula"
ufw route allow in on tun0 to 192.168.40.12 port 445 proto tcp comment "VPN → SMB FILE01"
ufw route allow in on tun0 to 192.168.40.12 port 21 proto tcp comment "VPN → FTP FILE01"
ufw route allow in on tun0 to 192.168.40.10 port 389 proto tcp comment "VPN → LDAP DC01"
ufw route allow in on tun0 to 192.168.40.10 port 88 proto tcp comment "VPN → Kerberos DC01"
ufw route allow in on tun0 to 192.168.40.10 port 636 proto tcp comment "VPN → LDAPS DC01"
ufw route allow in on tun0 to 192.168.40.10 port 53 proto tcp comment "VPN → DNS DC01"
```
<img width="694" height="369" alt="image" src="https://github.com/user-attachments/assets/63ba8cba-76a4-4c1e-a350-71e361ad86b8" />



#### Servidor Web (WEB01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow 80/tcp comment "HTTP"
ufw allow 443/tcp comment "HTTPS"
ufw allow from 192.168.40.0/24 to any port 3306 proto tcp comment "MySQL"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix"
ufw allow from 192.168.40.13 to any port 9102 proto tcp comment "Bacula"
```
<img width="709" height="290" alt="ufw_status" src="https://github.com/user-attachments/assets/1657f012-4eb4-4727-bbdb-a03e1cbedb45" />


#### Servidor Backup (BAK01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow from 192.168.40.0/24 to any port 9101 proto tcp comment "Bacula Director"
ufw allow from 192.168.40.0/24 to any port 9102 proto tcp comment "Bacula FD"
ufw allow from 192.168.40.0/24 to any port 9103 proto tcp comment "Bacula SD desde LAN"
ufw allow from 192.168.100.0/24 to any port 9103 proto tcp comment "Bacula SD desde DMZ"
ufw allow from 192.168.60.0/24 to any port 9103 proto tcp comment "Bacula SD desde SEC"
ufw allow from 192.168.70.0/24 to any port 9103 proto tcp comment "Bacula SD desde MON"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix Agent"
```
<img width="771" height="257" alt="ufw_status" src="https://github.com/user-attachments/assets/e8c73d49-710b-4768-812c-515e88cfdb81" />


#### Servidor Monitorización (MON01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow from 192.168.40.0/24 to any port 80 proto tcp comment "Zabbix Web LAN"
ufw allow from 192.168.40.0/24 to any port 3000 proto tcp comment "Grafana LAN"
ufw allow from 192.168.10.0/24 to any port 80 proto tcp comment "Zabbix Web Admin"
ufw allow from 192.168.10.0/24 to any port 3000 proto tcp comment "Grafana Admin"
ufw allow from 192.168.30.0/24 to any port 80 proto tcp comment "Zabbix Web IT"
ufw allow from 192.168.30.0/24 to any port 3000 proto tcp comment "Grafana IT"
ufw allow 10051/tcp comment "Zabbix Server active checks"
ufw allow from 192.168.40.13 to any port 9101:9104 proto tcp comment "Bacula"
```
<img width="828" height="277" alt="ufw_status" src="https://github.com/user-attachments/assets/3931f34c-d379-4c42-87f1-26ab6ca236b9" />


---

## 3.9 Conclusiones

> La implementación de seguridad sigue las mejores prácticas:

- ✅ **Defensa en profundidad:** Múltiples capas de seguridad
- ✅ **Mínimo privilegio:** Solo se conceden los permisos necesarios
- ✅ **Segmentación:** VLANs aíslan diferentes zonas de confianza
- ✅ **Monitorización:** IDS activo + logs de auditoría
- ✅ **Seguridad:** Hardening consistente de servidores
- ✅ **Actualizaciones:** Parches de seguridad automáticos

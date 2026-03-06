# 03 - Seguridad

## 3.1 Introducción

Este documento describe las medidas de seguridad implementadas en la infraestructura TechLogix siguiendo el principio de **defensa en profundidad** con múltiples capas de protección.

---

## 3.2 Arquitectura de Seguridad

### 3.2.1 Modelo de Defensa en Profundidad

```
┌─────────────────────────────────────────────────────────────────┐
│                        CAPA 1: PERÍMETRO                        │
│                     pfSense Firewall + NAT                      │
│                        Snort IDS/IPS                            │
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
                                │
┌─────────────────────────────────────────────────────────────────┐
│                       CAPA 5: DATOS                             │
│              Backup 3-2-1 + RAID + Cifrado                      │
│                   Permisos de Archivos                          │
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
| Pass | UDP | * | WAN Address | 1194 | OpenVPN NAT |

#### LAN (Servers - 192.168.40.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | * | * | LAN Address | 443,80,22 | Anti-Lockout |
| Pass | TCP/UDP | LAN net | * | 123 | NTP |
| Pass | TCP | LAN net | OPT3 | 10051 | Zabbix Agent|
| Pass | UDP | LAN net | * | 53 | DNS |
| Pass | TCP | LAN net | * | 80,443 | HTTP/S |
| Pass | TCP | 192.168.40.13 | LAN net | 9101-9104 | Bacula |
| Pass | TCP | 192.168.40.12 | OPT2 | 389,445,21 | LDAP,SMB,FTP |

#### OPT1 (DMZ - 192.168.100.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Block | * | OPT1 | LAN net | * | Block to Servers |
| Block | * | OPT1 | OPT2-6 | * | Block to VLANs |
| Pass | TCP | OPT1 | OPT3 | 10051 | Zabbix |
| Pass | TCP/UDP | OPT1 | * | 443 | HTTPS |
| Pass | UDP | OPT1 | * | 53 | DNS |
| Pass | TCP | OPT1 | * | 80 | HTTP |
| Block | * | OPT1 | * | * | Block resto |

#### OPT2 (Security/VPN - 192.168.60.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | TCP | OPT2 | OPT3 | 10051 | Zabbix Agent |
| Pass | TCP | 10.8.0.0/24 | 192.168.40.12 | 21 | VPN to FTP |
| Pass | TCP/UDP | * | * | 9101-9102 | Bacula |
| Pass | TCP/UDP | OPT2 | 192.168.40.10 | 389,636 | LDAP Auth |
| Pass | TCP | OPT2 | * | 80,443 | HTTP/S |
| Pass | UDP | OPT2 | * | 53 | DNS |
| Pass | UDP | * | * | 1194 | OpenVPN |
| Block | * | OPT2 | * | * | Block resto |

#### OPT3 (Monitoring - 192.168.70.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | TCP | OPT3 | LAN net | 10050 | Zabbix Polling |
| Pass | TCP | OPT3 | * | 587 | SMTP Alertas |
| Pass | TCP | OPT3 | OPT2 | 2222 | SSH Ansible SEC |
| Pass | TCP | OPT3 | OPT1 | 2222 | SSH Ansible DMZ |
| Pass | TCP | OPT3 | LAN net | 2222 | SSH Ansible |
| Pass | UDP | OPT3 | * | 53 | DNS |
| Pass | TCP | OPT3 | * | 80,443 | HTTP/S |
| Pass | TCP/UDP | OPT3 | * | 10050 | Zabbix |
| Block | * | OPT3 | * | * | Block resto |

#### OPT4 (Admin - 192.168.10.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | TCP/UDP | OPT4 | LAN net | 67-68 | DHCP |
| Block | * | OPT4 | OPT1 | * | Block DMZ |
| Pass | TCP | OPT4 | 192.168.40.10 | 88 | Kerberos |
| Pass | TCP | OPT4 | 192.168.40.10 | 53,389,445,464,636 | AD |
| Pass | TCP | OPT4 | 192.168.40.12 | 445 | SMB |
| Pass | TCP | OPT4 | OPT3 | 80,3000 | Zabbix/Grafana |
| Pass | UDP | OPT4 | * | 53 | DNS |
| Pass | TCP | OPT4 | * | 443 | HTTPS |
| Block | * | OPT4 | * | * | Block resto |

#### OPT5 (Producción) y OPT6 (IT)
Reglas similares a OPT4 con acceso restringido a AD y SMB.

#### WIFI_GUESTS (192.168.50.0/24)
| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Block | * | WIFI_GUESTS | 192.168.0.0/16 | * | Block privado |
| Block | * | WIFI_GUESTS | 10.0.0.0/8 | * | Block VPN |
| Pass | UDP | WIFI_GUESTS | * | 53 | DNS |
| Pass | TCP | WIFI_GUESTS | * | 80,443 | Internet |
| Block | * | WIFI_GUESTS | * | * | Block resto |

---

## 3.4 Sistema de Detección de Intrusiones (Snort)

### 3.4.1 Configuración General

- **Ubicación:** Integrado en pfSense
- **Modo:** IPS (bloqueo activo)
- **Interfaces monitorizadas:** WAN, LAN, OPT1 (DMZ), OPT2 (VPN)

### 3.4.2 Reglas Activadas (IPS Policy)

- Politicas IPS predefenidas en las siguientes interfaces:

| Interfaz | Politica |
|-----------|-------------|
| WAN | Connectivity |
| LAN | Balancend |
| DMZ | Balanced |
| OPT2 | Balanced |

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

### 3.5.3 Restricción de Acceso

Solo los miembros del grupo **GRP_IT** pueden conectarse via VPN. Otros usuarios del dominio son rechazados.

### 3.5.4 Certificados

| Archivo | Ubicación | Propósito |
|---------|-----------|-----------|
| ca.crt | /etc/openvpn/server/ | Autoridad Certificadora |
| server.crt | /etc/openvpn/server/ | Certificado servidor |
| server.key | /etc/openvpn/server/ | Clave privada servidor |
| david-vpn.crt | PKI | Certificado cliente |
| ta.key | /etc/openvpn/server/ | TLS Auth |

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

El acceso no autorizado está prohibido.
Todas las actividades en este sistema son monitorizadas y registradas.

Si no está autorizado, desconéctese inmediatamente.
********************************************************************************
```

### 3.6.8 Servicios Deshabilitados

```bash
systemctl disable avahi-daemon
systemctl disable cups
```

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
ufw allow 135/tcp comment "RPC"
ufw allow from 192.168.40.0/24 to any port 49152:65535 proto tcp comment "RPC Dynamic"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix"
ufw allow 9101:9104/tcp comment "Bacula"
```

#### Servidor de Archivos (FILE01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow 21/tcp comment "FTP"
ufw allow 40000:50000/tcp comment "FTP Pasivo"
ufw allow from 192.168.10.0/24 to any port 445 proto tcp comment "SMB Admin"
ufw allow from 192.168.10.0/24 to any port 139 proto tcp comment "NetBIOS Admin"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix"
ufw allow 9101:9104/tcp comment "Bacula"
```

#### Servidor VPN (SEC01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow 1194/udp comment "OpenVPN"
ufw allow in on tun0 comment "VPN Traffic"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix"
ufw allow 9101:9104/tcp comment "Bacula"
```

#### Servidor Web (WEB01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow 80/tcp comment "HTTP"
ufw allow 443/tcp comment "HTTPS"
ufw allow from 192.168.40.0/24 to any port 3306 proto tcp comment "MySQL"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix"
ufw allow from 192.168.40.13 to any port 9102 proto tcp comment "Bacula"
```

#### Servidor Backup (BAK01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow from 192.168.40.0/24 to any port 9101 proto tcp comment "Bacula Director"
ufw allow from 192.168.40.0/24 to any port 9102 proto tcp comment "Bacula FD"
ufw allow from 192.168.40.0/24 to any port 9103 proto tcp comment "Bacula SD"
ufw allow from 192.168.70.10 to any port 10050 proto tcp comment "Zabbix"
```

#### Servidor Monitorización (MON01)

```bash
ufw allow 2222/tcp comment "SSH"
ufw allow from 192.168.40.0/24 to any port 80 proto tcp comment "Zabbix Web"
ufw allow from 192.168.40.0/24 to any port 3000 proto tcp comment "Grafana"
ufw allow from 192.168.10.0/24 to any port 80 proto tcp comment "Zabbix Admin"
ufw allow from 192.168.10.0/24 to any port 3000 proto tcp comment "Grafana Admin"
ufw allow 10051/tcp comment "Zabbix Server"
```

---

## 3.8 Matriz de Acceso

### 3.8.1 Acceso entre VLANs

| Origen \ Destino | ADMIN | PROD | IT | SERVERS | DMZ | VPN | MON | WIFI |
|------------------|-------|------|-----|---------|-----|-----|-----|------|
| ADMIN | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ |
| PROD | ❌ | ✅ | ❌ | SMB | ❌ | ❌ | ❌ | ❌ |
| IT | ❌ | ❌ | ✅ | SSH,SMB | ❌ | ❌ | ✅ | ❌ |
| SERVERS | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | Zabbix | ❌ |
| DMZ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | Zabbix | ❌ |
| VPN | ❌ | ❌ | ❌ | FTP | ❌ | ✅ | ❌ | ❌ |
| MON | SSH | ❌ | ❌ | SSH,Zabbix | SSH | SSH | ✅ | ❌ |
| WIFI | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | Internet |

---

## 3.9 Conclusiones

La implementación de seguridad sigue las mejores prácticas:

- ✅ **Defensa en profundidad:** Múltiples capas de seguridad
- ✅ **Mínimo privilegio:** Solo se conceden los permisos necesarios
- ✅ **Segmentación:** VLANs aíslan diferentes zonas de confianza
- ✅ **Monitorización:** IDS activo + logs de auditoría
- ✅ **Automatización:** Hardening consistente via Ansible
- ✅ **Actualizaciones:** Parches de seguridad automáticos

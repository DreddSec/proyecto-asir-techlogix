# 05 - Monitorización

## 5.1 Introducción

Este documento describe el sistema de monitorización implementado en TechLogix utilizando **Zabbix** como plataforma principal y **Grafana** para visualización avanzada.

---

## 5.2 Arquitectura de Monitorización

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SRV-MON01                                   │
│                      192.168.70.10                                  │
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐              │
│  │   Zabbix    │    │   Zabbix    │    │   Grafana   │              │
│  │   Server    │◄───│    Web      │    │   :3000     │              │
│  │   :10051    │    │    :80      │    │             │              │
│  └──────┬──────┘    └─────────────┘    └──────┬──────┘              │
│         │                                     │                     │
│         │              MySQL                  │                     │
│         └──────────►  :3306  ◄────────────────┘                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ Polling (10050)
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐     ┌──────────────┐      ┌──────────────┐
│    DC01      │     │   FILE01     │      │   WEB01      │
│ Zabbix Agent │     │ Zabbix Agent │      │ Zabbix Agent │
│    :10050    │     │    :10050    │      │    :10050    │
└──────────────┘     └──────────────┘      └──────────────┘
```

---

## 5.3 Zabbix Server

### 5.3.1 Información General

| Parámetro | Valor |
|-----------|-------|
| Versión | 7.0.x LTS |
| Servidor | SRV-MON01 (192.168.70.10) |
| Puerto Server | 10051 |
| Puerto Web | 80 |
| Base de Datos | MySQL 8.0 |
| URL | http://192.168.70.10/zabbix |

### 5.3.3 Instalación

```bash
# Repositorio Zabbix
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu24.04_all.deb
dpkg -i zabbix-release_7.0-1+ubuntu24.04_all.deb
apt update

# Instalar componentes
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent

# Crear base de datos
mysql -u root -p
> create database zabbix character set utf8mb4 collate utf8mb4_bin;
> create user zabbix@localhost identified by 'password';
> grant all privileges on zabbix.* to zabbix@localhost;

# Importar esquema
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix

# Configurar y arrancar
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
```

---

## 5.4 Zabbix Agent

### 5.4.1 Hosts Monitorizados

| Host | IP | VLAN | Template | Estado |
|------|-----|------|----------|--------|
| SRV-DC01 | 192.168.40.10 | Servers | Linux by Zabbix agent | ✅ |
| SRV-DC02 | 192.168.40.14 | Servers | Linux by Zabbix agent | ✅ |
| SRV-FILE01 | 192.168.40.12 | Servers | Linux by Zabbix agent | ✅ |
| SRV-BAK01 | 192.168.40.13 | Servers | Linux by Zabbix agent | ✅ |
| SRV-WEB01 | 192.168.100.8 | DMZ | Linux by Zabbix agent | ✅ |
| SRV-SEC01 | 192.168.60.10 | Security | Linux by Zabbix agent | ✅ |
| SRV-MON01 | 192.168.70.10 | Monitoring | Linux by Zabbix agent | ✅ |

### 5.4.2 Configuración del Agente

**Archivo:** `/etc/zabbix/zabbix_agentd.conf`

```ini
# Servidor Zabbix
Server=192.168.70.10
ServerActive=192.168.70.10

# Identificación del host
Hostname=SRV-DC01

# Puerto del agente
ListenPort=10050

# Directorio de configuración adicional
Include=/etc/zabbix/zabbix_agentd.d/*.conf

# Logs
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=0

# Seguridad
AllowKey=system.run[*]
```

### 5.4.3 Instalación del Agente

```bash
# En cada servidor a monitorizar
apt install zabbix-agent

# Editar configuración
nano /etc/zabbix/zabbix_agentd.conf
# Cambiar: Server, ServerActive, Hostname

# Reiniciar
systemctl restart zabbix-agent
systemctl enable zabbix-agent
```

### 5.4.4 Verificación de Conectividad

```bash
# Desde MON01, verificar agente de DC01
zabbix_get -s 192.168.40.10 -k system.hostname

# Verificar que el agente está escuchando
ss -tlnp | grep 10050
```

---

## 5.5 Templates Utilizados

### 5.5.1 Linux by Zabbix agent

Template estándar que monitoriza:

| Categoría | Métricas |
|-----------|----------|
| CPU | Utilización, carga, procesos |
| Memoria | RAM utilizada, disponible, swap |
| Disco | Espacio usado, inodos, I/O |
| Red | Tráfico entrante/saliente, errores |
| Sistema | Uptime, usuarios logueados, procesos |

### 5.5.2 Items Monitorizados

| Item | Key | Descripción |
|------|-----|-------------|
| CPU utilization | system.cpu.util | % uso de CPU |
| Available memory | vm.memory.available | RAM disponible |
| Free disk space | vfs.fs.size[/,pfree] | % disco libre |
| Network in | net.if.in[eth0] | Bytes entrantes |
| System uptime | system.uptime | Tiempo encendido |

---

## 5.6 Triggers y Alertas

### 5.6.1 Triggers Predefinidos

| Trigger | Severidad | Condición |
|---------|-----------|-----------|
| CPU alta | Warning | CPU > 80% por 5 min |
| Memoria baja | Average | RAM disponible < 10% |
| Disco lleno | High | Espacio libre < 10% |
| Servidor caído | Disaster | Sin respuesta 3 min |
| Servicio caído | High | Proceso no encontrado |

### 5.6.2 Configuración de Alertas por Email

**Administration → Media types → Email:**

| Parámetro | Valor |
|-----------|-------|
| SMTP server | smtp.gmail.com |
| SMTP port | 587 |
| SMTP email | alertas@techlogix.com |
| Connection security | STARTTLS |
| Authentication | Username and password |

**Configurar usuario para recibir alertas:**

1. Administration → Users → Admin
2. Media → Add
3. Type: Email
4. Send to: admin@techlogix.com
5. Severity: Todas marcadas

---

## 5.7 Grafana

### 5.7.1 Información General

| Parámetro | Valor |
|-----------|-------|
| Versión | 10.x |
| Puerto | 3000 |
| URL | http://192.168.70.10:3000 |
| Usuario default | admin |
| Contraseña default | admin |

### 5.7.2 Instalación

```bash
# Añadir repositorio
apt install -y software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"

# Instalar
apt update
apt install grafana

# Arrancar
systemctl start grafana-server
systemctl enable grafana-server
```

### 5.7.3 Plugin de Zabbix

```bash
# Instalar plugin
grafana-cli plugins install alexanderzobnin-zabbix-app

# Reiniciar Grafana
systemctl restart grafana-server
```

### 5.7.4 Configuración del Datasource

1. Configuration → Plugins → Zabbix → Enable
2. Configuration → Data Sources → Add data source
3. Seleccionar **Zabbix**
4. Configurar:

| Campo | Valor |
|-------|-------|
| Name | Zabbix-TechLogix |
| URL | http://localhost/zabbix/api_jsonrpc.php |
| Username | Admin |
| Password | [contraseña de Zabbix] |

5. Save & Test

---

## 5.8 Dashboards

### 5.8.1 Dashboard Principal: Infrastructure Overview

**Paneles incluidos:**

| Panel | Tipo | Query |
|-------|------|-------|
| CPU - All Servers | Time series | Group: *, Host: *, Item: CPU utilization |
| Memory Usage | Gauge | Group: *, Host: *, Item: Available memory |
| Disk Space | Bar gauge | Group: *, Host: *, Item: Free disk space |
| Active Problems | Table | Mode: Problems |
| Server Status | Stat | Group: *, Host: *, Item: Agent availability |

### 5.8.2 Crear Panel de CPU

1. Dashboard → New Dashboard → Add new panel
2. Data source: Zabbix-TechLogix
3. Query:
   - Group: `/.*/` (todos)
   - Host: `/.*/` (todos)
   - Item: `CPU utilization`
4. Visualization: Time series
5. Title: "CPU Utilization - All Servers"
6. Apply

### 5.8.3 Crear Panel de Problemas

1. Add new panel
2. Query Mode: **Zabbix Problems**
3. Visualization: Table
4. Title: "Active Problems"
5. Apply

---

## 5.9 Ansible para Monitorización

### 5.9.1 Gestión Centralizada

SRV-MON01 también funciona como servidor Ansible para gestión de configuración de todos los servidores.

**Estructura:**

```
/home/ansible/
├── hosts.ini           # Inventario
├── securizacion.yml    # Playbook de hardening
├── ufw_granular.yml    # Playbook de firewall
└── bacula_rclone.yml   # Playbook de backup
```

### 5.9.2 Inventario

**Archivo:** `/home/ansible/hosts.ini`

```ini
[all:vars]
ansible_user=sysadmin
ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa_ansible
ansible_port=2222
ansible_become=yes

[domain_controllers]
dc01 ansible_host=192.168.40.10
dc02 ansible_host=192.168.40.14

[file_servers]
file01 ansible_host=192.168.40.12

[backup_servers]
bak01 ansible_host=192.168.40.13

[web_servers]
web01 ansible_host=192.168.100.8

[security_servers]
sec01 ansible_host=192.168.60.10

[monitoring]
mon01 ansible_host=192.168.70.10

[servers:children]
domain_controllers
file_servers
backup_servers
web_servers
security_servers
```

### 5.9.3 Ejecución de Playbooks

```bash
# Verificar conectividad
ansible all -i hosts.ini -m ping --ask-become-pass

# Ejecutar playbook
ansible-playbook -i hosts.ini securizacion.yml --ask-become-pass
```

---

## 5.10 Métricas y KPIs

### 5.10.1 KPIs de Infraestructura

| KPI | Umbral Warning | Umbral Critical |
|-----|----------------|-----------------|
| CPU Utilization | > 80% | > 95% |
| Memory Usage | > 85% | > 95% |
| Disk Usage | > 80% | > 90% |
| Network Utilization | > 70% | > 90% |
| Response Time | > 1s | > 5s |

### 5.10.2 Disponibilidad Objetivo

| Servicio | SLA Objetivo |
|----------|--------------|
| Active Directory | 99.9% |
| Servidor de Archivos | 99.5% |
| Servidor Web | 99.0% |
| VPN | 99.0% |
| Backup | 99.0% |

---

## 5.11 Troubleshooting

### 5.11.1 Problemas Comunes

**Agente no responde:**
```bash
# Verificar servicio
systemctl status zabbix-agent

# Verificar conectividad
telnet 192.168.70.10 10050

# Verificar firewall
ufw status | grep 10050
```

**No hay datos en Grafana:**
```bash
# Verificar datasource
curl -X POST http://localhost/zabbix/api_jsonrpc.php \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"apiinfo.version","params":{},"id":1}'
```

**Alertas no se envían:**
```bash
# Verificar logs de Zabbix
tail -f /var/log/zabbix/zabbix_server.log

# Probar email
echo "Test" | mail -s "Zabbix Test" admin@techlogix.com
```

---

## 5.12 Conclusiones

El sistema de monitorización implementado proporciona:

- ✅ **Visibilidad completa:** Todos los servidores monitorizados
- ✅ **Métricas en tiempo real:** CPU, RAM, Disco, Red
- ✅ **Alertas proactivas:** Email ante problemas
- ✅ **Dashboards visuales:** Grafana con datos de Zabbix
- ✅ **Automatización:** Ansible desde el mismo servidor
- ✅ **Escalabilidad:** Fácil añadir nuevos hosts y métricas

# 05 - Monitorización

## 5.1 Introducción

> Este documento describe el sistema de monitorización implementado en TechLogix utilizando **Zabbix** como plataforma principal y **Grafana** para visualización avanzada.

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
# Instalar componentes
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent

# Crear base de datos
mysql -u root -p
> create database zabbix character set utf8mb4 collate utf8mb4_bin;
> create user zabbix@localhost identified by '[PASSWORD]';
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
<img width="1915" height="495" alt="zabbix_web_equipos" src="https://github.com/user-attachments/assets/288c13c9-2636-444e-890a-524a4caacdf2" />



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

> Ejemplo de una alerta por email por uso elevado de CPU (CPU > 80%):
<img width="1913" height="884" alt="email_alert" src="https://github.com/user-attachments/assets/e9e78906-f9d1-4a3c-a499-2e0b7370bd72" />

---

## 5.7 Grafana

### 5.7.1 Información General

| Parámetro | Valor |
|-----------|-------|
| Versión | 10.x |
| Puerto | 3000 |
| URL | http://192.168.70.10:3000 |

### 5.7.2 Plugin de Zabbix

```bash
# Instalar plugin
grafana-cli plugins install alexanderzobnin-zabbix-app

# Reiniciar Grafana
systemctl restart grafana-server
```
---

## 5.8 Dashboards

### 5.8.1 Dashboard Principal: Infrastructure Overview

**Paneles incluidos:**

| Panel | Tipo | Query |
|-------|------|-------|
| CPU - All Servers | Time series | Group: *, Host: *, Item: CPU utilization |
| Memory Usage | Gauge | Group: *, Host: *, Item: Available memory |
| Disk Space | Bar gauge | Group: *, Host: *, Item: Free disk space |
<img width="1921" height="932" alt="grafana_dashboard" src="https://github.com/user-attachments/assets/3ce43929-5e9d-42b3-92a9-901e0e63d011" />

---

## 5.9 Métricas y KPIs

### 5.9.1 KPIs de Infraestructura

| KPI | Umbral Warning | Umbral Critical |
|-----|----------------|-----------------|
| CPU Utilization | > 80% | > 95% |
| Memory Usage | > 85% | > 95% |
| Disk Usage | > 80% | > 90% |
| Network Utilization | > 70% | > 90% |
| Response Time | > 1s | > 5s |

---

## 5.10 Troubleshooting

### 5.10.1 Problemas Comunes

**Agente no responde:**
```bash
# Verificar servicio
systemctl status zabbix-agent

# Verificar conectividad
nc -nv 192.168.70.10 10050

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

> El sistema de monitorización implementado proporciona:

- ✅ **Visibilidad completa:** Todos los servidores monitorizados
- ✅ **Métricas en tiempo real:** CPU, RAM, Disco, Red
- ✅ **Alertas proactivas:** Alerta mediante email ante problemas
- ✅ **Dashboards visuales:** Grafana con datos de Zabbix
- ✅ **Escalabilidad:** Fácil añadir nuevos hosts y métricas

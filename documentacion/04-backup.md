# 04 - Sistema de Backup

## 4.1 Introducción

Este documento describe el sistema de backup implementado en TechLogix siguiendo la **estrategia 3-2-1**, reconocida como una de las mejores prácticas en protección de datos.

---

## 4.2 Estrategia 3-2-1

### 4.2.1 Concepto

La regla 3-2-1 establece:
- **3** copias de los datos (original + 2 backups)
- **2** tipos de almacenamiento diferentes
- **1** copia offsite (fuera del sitio)

### 4.2.2 Implementación en TechLogix

```
┌─────────────────┐
│  DATOS ORIGEN   │
│  (Servidores)   │
│                 │
│  • /etc         │
│  • /home        │
│  • /var/log     │
│  • /var/www     │
└────────┬────────┘
         │
         │ Bacula File Daemon
         │ (Puerto 9102)
         ▼
┌─────────────────┐
│     COPIA 1     │
│   (Original)    │
│                 │
│                 │
│                 │
└────────┬────────┘
         │
         │ Red interna
         │
         ▼
┌─────────────────┐
│    COPIA 2      │
│    (Local)      │
│                 │
│   SRV-BAK01     │
│    RAID 5       │
│ /raid5/bacula   │
└────────┬────────┘
         │
         │ Rclone (Internet)
         │
         ▼
┌─────────────────┐
│     COPIA 3     │
│     (Cloud)     │
│                 │
│  Google Drive   │
│     gdrive:     │
│ backups/bacula  │
└─────────────────┘
```

---

## 4.3 Infraestructura de Backup

### 4.3.1 Servidor de Backup (SRV-BAK01)

| Componente | Especificación |
|------------|----------------|
| IP | 192.168.40.13 |
| Sistema Operativo | Ubuntu Server 24.04 |
| RAM | 2 GB |
| Almacenamiento Sistema | 200 GB |
| Almacenamiento RAID | 5 x 40 GB (RAID 5) |
| Capacidad Útil RAID | ~160 GB |

### 4.3.2 Configuración RAID 5

```bash
# Verificar estado del RAID
cat /proc/mdstat

# Ejemplo de salida
md0 : active raid5 sde1[3] sdc1[1] sda3[0] sdb1[2] sdd1[5]
      157151232 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
```

**Punto de montaje:**
```bash
/dev/md0 on /raid5 type ext4 (rw,relatime)
```

**Estructura de directorios:**
```
/raid5/
├── bacula-backups/    # Volúmenes de Bacula
└── bacula-restore/    # Restauraciones temporales
```

---

## 4.4 Bacula

### 4.4.1 Arquitectura

```
                    ┌─────────────────┐
                    │ Bacula Director │
                    │   (BAK01)       │
                    │   Puerto 9101   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────┐
       │   DC01   │   │  FILE01  │   │  WEB01   │
       │ Bacula   │   │ Bacula   │   │ Bacula   │
       │   FD     │   │   FD     │   │   FD     │
       │  :9102   │   │  :9102   │   │  :9102   │
       └──────────┘   └──────────┘   └──────────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Bacula Storage  │
                    │   Daemon        │
                    │   (BAK01)       │
                    │   Puerto 9103   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    RAID 5       │
                    │ /raid5/bacula-  │
                    │    backups/     │
                    └─────────────────┘
```

### 4.4.2 Componentes

| Componente | Servidor | Puerto | Función |
|------------|----------|--------|---------|
| Director | BAK01 | 9101 | Orquesta los backups |
| Storage Daemon | BAK01 | 9103 | Escribe en almacenamiento |
| File Daemon | Todos | 9102 | Envía datos de cada servidor |
| Catalog | BAK01 (MySQL) | 3306 | Base de datos de metadatos |

### 4.4.3 Configuración del Director

**Archivo:** `/etc/bacula/bacula-dir.conf`

```ini
Director {
  Name = bacula-dir
  DIRport = 9101
  QueryFile = "/etc/bacula/query.sql"
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/run/bacula"
  Maximum Concurrent Jobs = 20
  Password = "[PASSWORD]"
  Messages = Daemon
  DirAddress = 192.168.40.13
}

JobDefs {
  Name = "DefaultJob"
  Type = Backup
  Level = Incremental
  Client = dc02-fd
  FileSet = "Full Set"
  Schedule = "WeeklyCycle"
  Storage = File
  Messages = Standard
  Pool = Default
  Priority = 10
  Accurate = yes
  Spool Data = yes
  Full Backup Pool = Full
  Incremental Backup Pool = Incremental
  Differential Backup Pool = Differential
  Write Bootstrap = "/var/lib/bacula/%c.bsr"

  RunScript {
    Command = "/usr/local/bin/sync-to-gdrive.sh"
    RunsWhen = After
    RunsOnClient = No
 }
}

# Ejemplo de un Job al servidor DC01 (resto de servidores se aplica lo mismo)
Job {
  Name = "BackupDC01"
  JobDefs = "DefaultJob"
  Client = dc01-fd
}

# 
Job {
  Name = "RestoreFiles"
  Type = Restore
  Client = dc02-fd
  FileSet = "Full Set"
  Storage = File
  Pool = Default
  Messages = Standard
  Where = /raid5/bacula-restore
}


# Catálogo MySQL
Catalog {
  Name = MyCatalog
  dbname = "bacula"
  dbuser = "bacula"
  dbpassword = "[DB_PASSWORD]"
}

# Programación de Backups
Schedule {
  Name = "WeeklyCycle"
  Run = Full 1st sun at 23:00
  Run = Differential 2nd-5th sun at 23:00
  Run = Incremental mon-sat at 23:00
}

# FileSet - Qué respaldar
FileSet {
  Name = "Full Set"
  Include {
    Options {
      signature = MD5
      compression = GZIP
    }
    File = /etc
    File = /home
    File = /var/log
    File = /var/www
  }
  Exclude {
    File = /proc
    File = /tmp
    File = /sys
    File = /dev
  }
}

# Job para DC01
Job {
  Name = "BackupDC01"
  Type = Backup
  Level = Incremental
  Client = dc01-fd
  FileSet = "Full Set"
  Schedule = "WeeklyCycle"
  Storage = File1
  Pool = Default
  Priority = 10
  Write Bootstrap = "/var/lib/bacula/%c.bsr"
}
# Jobs similares para DC02, FILE01, WEB01, SEC01, MON01...

# Pools para definir y agrupar volumenes de almacenamiento
Pool {
  Name = Full
  Pool Type = Backup
  Recycle = yes
  AutoPrune = yes
  Volume Retention = 365 days
  Maximum Volume Bytes = 50G
  Maximum Volumes = 100
}
 
Pool {
  Name = Differential
  Pool Type = Backup
  Recycle = yes
  AutoPrune = yes
  Volume Retention = 90 days
  Maximum Volume Bytes = 20G
  Maximum Volumes = 100
}

Pool {
  Name = Incremental
  Pool Type = Backup
  Recycle = yes
  AutoPrune = yes
  Volume Retention = 30 days
  Maximum Volume Bytes = 10G
  Maximum Volumes = 100
}
```

### 4.4.4 Configuración del File Daemon (Cliente)

**Archivo en cada servidor:** `/etc/bacula/bacula-fd.conf`

```ini
Director {
  Name = bacula-dir
  Password = "BaculaSuperPass123"
}

FileDaemon {
  Name = srv-bak01-fd
  FDport = 9102
  WorkingDirectory = /var/lib/bacula
  Pid Directory = /run/bacula
  Maximum Concurrent Jobs = 20
  Plugin Directory = /usr/lib/bacula
  FDAddress = 192.168.40.13
}

Messages {
  Name = Standard
  director = bacula-dir = all
}
```

### 4.4.5 Configuración del Storage Daemon

**Archivo:** `/etc/bacula/bacula-sd.conf`

```ini
  GNU nano 8.6                                                                                      bacula-sd.conf                                                                                               
Storage {
  Name = bacula-sd
  SDPort = 9103
  WorkingDirectory = "/var/lib/bacula"
  Pid Directory = "/run/bacula"
  Maximum Concurrent Jobs = 20
  SDAddress = 192.168.40.13
}

Director {
  Name = bacula-dir
  Password = "[PASSWORD]"
}

Director {
  Name = bacula-mon
  Password = "[PASSWORD]"
  Monitor = yes
}

Device {
  Name = FileStorage
  Media Type = File
  Archive Device = /raid5/bacula-backups
  LabelMedia = yes
  Random Access = Yes
  AutomaticMount = yes
  RemovableMedia = no
  AlwaysOpen = no
}

Messages {
  Name = Standard
  director = bacula-dir = all
}
```

### 4.4.6 Pools de Almacenamiento

| Pool | Tipo | Retención | Descripción |
|------|------|-----------|-------------|
| Full | Full | 30 días | Backups completos mensuales |
| Default | Incremental | 7 días | Backups diarios |
| Differential | Differential | 14 días | Backups semanales |

### 4.4.7 Programación de Backups

| Día | Tipo | Hora |
|-----|------|------|
| 1er Domingo del mes | Full | 23:00 |
| Domingos 2-5 | Diferencial | 23:00 |
| Lunes a Sábado | Incremental | 23:00 |

### 4.4.8 Comandos de Administración

```bash
# Acceder a la consola de Bacula
sudo bconsole

# Dentro de bconsole:
*status director          # Estado del director
*status client            # Estado de clientes
*list jobs                # Listar trabajos
*run job=BackupDC01       # Ejecutar backup manual
*restore                  # Iniciar restauración
*messages                 # Ver mensajes
```

---

## 4.5 Sincronización a la Nube (Rclone)

### 4.5.1 Configuración

**Archivo:** `/root/.config/rclone/rclone.conf`

```ini
[gdrive]
type = drive
scope = drive
token = {"access_token":"...","token_type":"Bearer","expiry":"..."}
```

### 4.5.2 Script de Sincronización

**Archivo:** `/usr/local/bin/sync-to-gdrive.sh`

```bash
#!/bin/bash

LOG_FILE="/var/log/rclone-sync.log"
BACKUP_DIR="/raid5/bacula-backups"
REMOTE="gdrive:backups/bacula"

echo "=== Inicio de sincronización: $(date) ===" >> "$LOG_FILE"

# Verificar configuración
if ! rclone listremotes | grep -q "gdrive:"; then
    echo "ERROR: Rclone no configurado" >> "$LOG_FILE"
    exit 1
fi

# Sincronizar
rclone sync "$BACKUP_DIR" "$REMOTE" \
    --transfers 4 \
    --checkers 8 \
    --contimeout 60s \
    --timeout 300s \
    --retries 3 \
    --log-file="$LOG_FILE" \
    --log-level INFO

if [ $? -eq 0 ]; then
    echo "Sincronización exitosa: $(date)" >> "$LOG_FILE"
else
    echo "ERROR en sincronización: $(date)" >> "$LOG_FILE"
    exit 1
fi

echo "=== Fin de sincronización ===" >> "$LOG_FILE"
```

### 4.5.3 Automatización (Cron)

**Crontab de root:**

```bash
# Sincronización diaria a las 3:00 AM
0 3 * * * /usr/local/bin/sync-to-gdrive.sh
```

### 4.5.4 Comandos de Verificación

```bash
# Listar contenido en Google Drive
sudo rclone ls gdrive:backups/bacula

# Verificar espacio usado
sudo rclone size gdrive:backups/bacula

# Sincronización manual
sudo rclone sync /raid5/bacula-backups/ gdrive:backups/bacula --progress
```

---

## 4.6 Procedimientos de Restauración

### 4.6.1 Restauración desde Bacula

```bash
# 1. Acceder a bconsole
sudo bconsole

# 2. Iniciar restauración
*restore

# 3. Seleccionar opciones:
#    - Cliente a restaurar
#    - Fecha del backup
#    - Archivos a restaurar
```

### 4.6.2 Restauración desde Google Drive

**Script:** `/usr/local/bin/emergency-restore.sh`

```bash
#!/bin/bash

if [ "$#" -ne 2 ]; then
    echo "Uso: $0 <nombre_servidor> <ruta_destino>"
    exit 1
fi

SERVIDOR=$1
DESTINO=$2

echo "Restaurando: $SERVIDOR a $DESTINO"

rclone copy "gdrive:backups/bacula" "$DESTINO" \
    --progress \
    --transfers 4

echo "Archivos descargados. Use bconsole para restaurar."
```

---

## 4.7 Monitorización de Backups

### 4.7.1 Verificación de Integridad

**Script:** `/usr/local/bin/verify-backups.sh`

```bash
#!/bin/bash

LOG_FILE="/var/log/backup-verification.log"

echo "=== Verificación: $(date) ===" >> "$LOG_FILE"

# Verificar espacio RAID
DISK_USAGE=$(df -h /raid5 | tail -1 | awk '{print $5}' | sed 's/%//')
echo "Uso RAID5: ${DISK_USAGE}%" >> "$LOG_FILE"

if [ "$DISK_USAGE" -gt 85 ]; then
    echo "ADVERTENCIA: Disco >85%" >> "$LOG_FILE"
fi

# Contar volúmenes
BACKUP_COUNT=$(find /raid5/bacula-backups -type f -name "*.vol" | wc -l)
echo "Volúmenes: $BACKUP_COUNT" >> "$LOG_FILE"

# Verificar RAID
if grep -q "UU" /proc/mdstat; then
    echo "RAID5: OK" >> "$LOG_FILE"
else
    echo "CRÍTICO: Problema RAID5" >> "$LOG_FILE"
fi
```

## 4.8 Volúmenes de Backup

### 4.8.1 Estado Actual

```bash
ls -la /raid5/bacula-backups/
```

| Volumen | Tamaño | Fecha | Pool |
|---------|--------|-------|------|
| Vol-0001 | 222 B | Feb 3 | Default |
| Vol-Diff-0001 | 232 B | Feb 3 | Differential |
| Vol-Full-0001 | 9.6 MB | Feb 3 | Full |
| Vol-Inc-0001 | 230 B | Feb 3 | Default |

---

## 4.9 Conclusiones

El sistema de backup implementado cumple con:

- ✅ **Estrategia 3-2-1:** 3 copias, 2 medios, 1 offsite
- ✅ **Automatización:** Backups programados sin intervención
- ✅ **Redundancia local:** RAID 5 protege contra fallo de 1 disco
- ✅ **Offsite:** Google Drive para recuperación ante desastres
- ✅ **Granularidad:** Full, Diferencial, Incremental
- ✅ **Monitorización:** Alertas y verificación de integridad

# 06 - Automatización con Ansible

## 6.1 Introducción

Este documento describe la implementación de **Ansible** como herramienta de automatización para la gestión centralizada de configuraciones en la infraestructura TechLogix.

**Ansible** permite ejecutar tareas de forma consistente en múltiples servidores simultáneamente, garantizando configuraciones homogéneas y reduciendo errores humanos.

---

## 6.2 Arquitectura

### 6.2.1 Servidor de Control

| Parámetro | Valor |
|-----------|-------|
| Servidor | SRV-MON01 |
| IP | 192.168.70.10 |
| Usuario Ansible | ansible |
| Directorio de trabajo | /home/ansible |

### 6.2.2 Nodos Gestionados

| Servidor | IP | Usuario SSH | Puerto |
|----------|-----|-------------|--------|
| SRV-DC01 | 192.168.40.10 | sysadmin | 2222 |
| SRV-DC02 | 192.168.40.14 | sysadmin | 2222 |
| SRV-FILE01 | 192.168.40.12 | sysadmin | 2222 |
| SRV-BAK01 | 192.168.40.13 | sysadmin | 2222 |
| SRV-WEB01 | 192.168.100.8 | sysadmin | 2222 |
| SRV-SEC01 | 192.168.60.10 | sysadmin | 2222 |

### 6.2.3 Diagrama de Conexión

```
                    ┌─────────────────┐
                    │    SRV-MON01    │
                    │  (Ansible Host) │
                    │  192.168.70.10  │
                    └────────┬────────┘
                             │
                             │ SSH (Puerto 2222)
                             │ Clave RSA
                             │
        ┌────────────────────┼────────────────────┐
        │          │         │         │          │
        ▼          ▼         ▼         ▼          ▼
    ┌───────┐  ┌───────┐ ┌───────┐ ┌───────┐  ┌───────┐
    │ DC01  │  │ DC02  │ │FILE01 │ │ BAK01 │  │ WEB01 │
    └───────┘  └───────┘ └───────┘ └───────┘  └───────┘
```

---

## 6.3 Configuración Inicial

### 6.3.1 Instalación de Ansible

```bash
# En SRV-MON01
sudo apt update
sudo apt install ansible -y

# Verificar instalación
ansible --version
```

### 6.3.2 Crear Usuario Ansible

```bash
# Crear usuario en MON01
sudo useradd -m -s /bin/bash ansible
sudo passwd ansible

# Generar clave SSH
sudo -u ansible ssh-keygen -t rsa -b 4096 -f /home/ansible/.ssh/id_rsa_ansible -N ""
```

## 5.9 Ansible para Monitorización

### 5.9.1 Gestión Centralizada

SRV-MON01 también funciona como servidor **Ansible** para gestión de configuración de todos los servidores.

**Estructura:**

```
/home/ansible/
├── hosts.ini           # Inventario
├── securizacion.yml    # Playbook de hardening
├── secrets.yml         # Proteger datos sensibles
└── bacula_rclone.yml   # Playbook de backup
```

### 6.3.3 Distribuir Clave SSH

```bash
# Copiar clave a cada servidor (ejecutar para cada uno)
sudo -u ansible ssh-copy-id -i /home/ansible/.ssh/id_rsa_ansible.pub -p 2222 sysadmin@IP_SERVIDOR
```

---

## 6.4 Inventario

### 6.4.1 Archivo hosts.ini

```ini
[all:vars]
ansible_user=sysadmin
ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa_ansible
ansible_port=2222
ansible_become=yes
ansible_become_method=sudo

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

[servers:children]
domain_controllers
file_servers
backup_servers
web_servers
security_servers
```

### 6.4.2 Verificar Conectividad

```bash
# Desde usuario ansible en MON01
ansible all -i hosts.ini -m ping -e @secrets.yml --ask-become-pass --ask-vault-pass
```
<img width="533" height="693" alt="ansible_ping" src="https://github.com/user-attachments/assets/c61568bf-e950-4075-91a1-46805553e78b" />


---

## 6.5 Playbooks Implementados

### 6.5.1 Playbook de Securización (hardening.yml)

Este playbook aplica hardening a todos los servidores de forma automatizada.

**Tareas incluidas:**

| Tarea | Descripción |
|-------|-------------|
| Actualizar sistema | apt update && apt upgrade |
| Instalar paquetes seguridad | fail2ban, ufw, auditd, libpam-pwquality |
| Hardening SSH | Puerto 2222, deshabilitar root, solo claves |
| Permisos archivos críticos | /etc/shadow, /etc/passwd |
| Políticas contraseñas | PAM con 12 caracteres mínimo |
| Configurar auditd | Monitorizar cambios en archivos críticos |
| Fail2ban | Protección contra fuerza bruta |
| Actualizaciones automáticas | unattended-upgrades |
| Banner legal | Mensaje de advertencia en SSH |
| Deshabilitar servicios | avahi-daemon, cups |

**Archivo: hardening.yml**

```yaml
---
- name: Securización y Hardening de Servidores Ubuntu
  hosts: servers
  become: yes
  vars:
    ssh_port: 2222
    
  tasks:
    # 1. ACTUALIZAR SISTEMA
    - name: Actualizar cache de paquetes
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Actualizar todos los paquetes
      apt:
        upgrade: dist
        autoremove: yes
        autoclean: yes

    # 2. INSTALAR PAQUETES DE SEGURIDAD
    - name: Instalar paquetes de seguridad
      apt:
        name:
          - fail2ban
          - ufw
          - auditd
          - libpam-pwquality
          - unattended-upgrades
          - apt-listchanges
          - ntp
        state: present

    # 3. HARDENING SSH
    - name: Configurar SSH - Deshabilitar root login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
        state: present

    - name: Configurar SSH - Deshabilitar autenticación por contraseña
      lineinfile:
       path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
        state: present

    - name: Configurar SSH - Deshabilitar autenticación por contraseña
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present

    - name: Configurar SSH - Cambiar puerto
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?Port'
        line: "Port {{ ssh_port }}"
        state: present

    - name: Configurar SSH - Deshabilitar X11 Forwarding
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?X11Forwarding'
        line: 'X11Forwarding no'
        state: present

    - name: Configurar SSH - MaxAuthTries
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?MaxAuthTries'
        line: 'MaxAuthTries 3'
        state: present

   - name: Reiniciar SSH
      systemd:
        name: ssh
        state: restarted

    # 4. PERMISOS DE ARCHIVOS CRÍTICOS
    - name: Configurar permisos /etc/shadow
      file:
        path: /etc/shadow
        mode: '0600'
        owner: root
        group: root

    - name: Configurar permisos /etc/gshadow
      file:
        path: /etc/gshadow
        mode: '0600'
        owner: root
        group: root

    - name: Configurar permisos /etc/passwd
      file:
        path: /etc/passwd
        mode: '0644'
        owner: root
        group: root

    - name: Configurar permisos /etc/group
      file:
        path: /etc/group
        mode: '0644'
        owner: root
        group: root


    # 5. POLÍTICAS DE CONTRASEÑAS (PAM)
    - name: Configurar calidad de contraseñas - minlen
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?\s*minlen'
        line: 'minlen = 12'
        state: present

    - name: Configurar calidad de contraseñas - minclass
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?\s*minclass'
        line: 'minclass = 3'
        state: present

    - name: Configurar calidad de contraseñas - ucredit
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?\s*ucredit'
        line: 'ucredit = -1'
        state: present

    - name: Configurar calidad de contraseñas - dcredit
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?\s*dcredit'
        line: 'dcredit = -1'
        state: present

    - name: Configurar calidad de contraseñas - ocredit
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?\s*ocredit'
        line: 'ocredit = -1'
        state: present

   # 6. CONFIGURAR AUDITD
    - name: Crear reglas de auditd para autenticación
      blockinfile:
        path: /etc/audit/rules.d/auth.rules
        create: yes
        block: |
          -w /etc/passwd -p wa -k passwd_changes
          -w /etc/group -p wa -k group_changes
          -w /etc/shadow -p wa -k shadow_changes
          -w /etc/gshadow -p wa -k gshadow_changes
          -w /etc/sudoers -p wa -k sudoers_changes
          -w /var/log/auth.log -p wa -k auth_log_changes

    - name: Crear reglas de auditd para red
      blockinfile:
        path: /etc/audit/rules.d/network.rules
        create: yes
        block: |
          -a always,exit -F arch=b64 -S socket -S connect -k network_connections
          -w /etc/hosts -p wa -k hosts_changes
          -w /etc/network/ -p wa -k network_changes

    - name: Reiniciar auditd
      systemd:
        name: auditd
        state: restarted

   # 7. CONFIGURAR FAIL2BAN
    - name: Crear configuración local de Fail2ban
      copy:
        dest: /etc/fail2ban/jail.local
        content: |
          [DEFAULT]
          bantime = 3600
          findtime = 600
          maxretry = 3
          
          [sshd]
          enabled = true
          port = {{ ssh_port }}
          logpath = /var/log/auth.log

    - name: Reiniciar Fail2ban
      systemd:
        name: fail2ban
        state: restarted
        enabled: yes

    # 8. ACTUALIZACIONES AUTOMÁTICAS
    - name: Configurar actualizaciones automáticas de seguridad
      copy:
        dest: /etc/apt/apt.conf.d/50unattended-upgrades
        content: |
          Unattended-Upgrade::Allowed-Origins {
              "${distro_id}:${distro_codename}-security";
          };
          Unattended-Upgrade::AutoFixInterruptedDpkg "true";
          Unattended-Upgrade::MinimalSteps "true";
          Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
          Unattended-Upgrade::Remove-Unused-Dependencies "true";
          Unattended-Upgrade::Automatic-Reboot "false";

    - name: Habilitar actualizaciones automáticas
      copy:
        dest: /etc/apt/apt.conf.d/20auto-upgrades
        content: |
          APT::Periodic::Update-Package-Lists "1";
          APT::Periodic::Unattended-Upgrade "1";
          APT::Periodic::AutocleanInterval "7";

    # 9. CONFIGURAR NTP
    - name: Habilitar y arrancar NTP
      systemd:
        name: ntp
        state: started
        enabled: yes

    # 10. LÍMITES DE RECURSOS (ULIMITS)
    - name: Configurar ulimits para seguridad
      blockinfile:
        path: /etc/security/limits.conf
        block: |
          * soft nofile 65535
          * hard nofile 65535
          * soft nproc 4096
          * hard nproc 4096


    # 10. LÍMITES DE RECURSOS (ULIMITS)
    - name: Configurar ulimits para seguridad
      blockinfile:
        path: /etc/security/limits.conf
        block: |
          * soft nofile 65535
          * hard nofile 65535
          * soft nproc 4096
          * hard nproc 4096

    # 11. BANNER LEGAL (MOTD)
    - name: Crear banner legal de acceso
      copy:
        dest: /etc/motd
        content: |
          ********************************************************************************
          ADVERTENCIA: Este sistema es de uso exclusivo para personal autorizado.
          
          El acceso no autorizado está prohibido y será perseguido legalmente.
          Todas las actividades en este sistema son monitorizadas y registradas.
          
          Si no está autorizado, desconéctese inmediatamente.
          ********************************************************************************

    # 12. DESHABILITAR SERVICIOS INNECESARIOS
    - name: Deshabilitar servicios innecesarios
      systemd:
        name: "{{ item }}"
        state: stopped
        enabled: no
      loop:
        - avahi-daemon
        - cups
      ignore_errors: yes

  handlers:
    - name: restart ssh
      systemd:
        name: ssh
        state: restarted
```

**Ejecución:**

```bash
ansible-playbook -i hosts.ini hardening.yml -e @secrets.yml --ask-become-pass --ask-vault-pass
```
<img width="1308" height="781" alt="ansible_hardniung_yml_1" src="https://github.com/user-attachments/assets/92d773d9-3c74-4e50-8aef-5f3cdf877223" />

---

### 6.5.3 Playbook de Backup (bacula_rclone.yml)

Este playbook configura los agentes de Bacula en todos los servidores.

**Archivo: bacula_rclone.yml**

```yaml
---
# INSTALACIÓN DE BACULA CLIENT EN TODOS LOS SERVIDORES
- name: Instalar Bacula Client en todos los servidores
  hosts: servers
  become: yes
  vars:
    bacula_director_password: [DIR_PASSWORD]
    bacula_server_ip: "192.168.40.13"
    
  tasks:
    - name: Instalar Bacula File Daemon
      apt:
        name: bacula-fd
        state: present
        update_cache: yes

    - name: Configurar Bacula File Daemon
      template:
        src: bacula-fd.conf.j2
        dest: /etc/bacula/bacula-fd.conf
        mode: '0640'
        owner: bacula
        group: bacula
      notify: restart bacula-fd

    - name: Habilitar y arrancar Bacula FD
      systemd:
        name: bacula-fd
        state: started
        enabled: yes

  handlers:
    - name: restart bacula-fd
      systemd:
        name: bacula-fd
        state: restarted

# CONFIGURACIÓN DEL SERVIDOR BACULA
- name: Configurar Servidor Bacula en BAK01
  hosts: backup_servers
  become: yes
  vars:
    bacula_db_password: [DB_PASWORD]
    bacula_director_password: [DIR_PASSWORD]
    
  tasks:
    - name: Instalar componentes Bacula Server
      apt:
        name:
          - bacula-client
          - bacula-console
          - mysql-server
          - python3-mysqldb
        state: present
        update_cache: yes

    - name: Crear base de datos MySQL para Bacula
      mysql_db:
        name: bacula
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Crear usuario MySQL para Bacula
      mysql_user:
        name: bacula
        password: "{{ bacula_db_password }}"
        priv: "bacula.*:ALL"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock


    - name: Importar esquema de Bacula
      mysql_db:
        name: bacula
        state: import
        target: /usr/share/bacula-director/make_mysql_tables
        login_unix_socket: /var/run/mysqld/mysqld.sock
      ignore_errors: yes

    - name: Crear directorio de backups en RAID
      file:
        path: /raid5/bacula-backups
        state: directory
        owner: bacula
        group: bacula
        mode: '0750'

    - name: Crear directorio de restore
      file:
        path: /raid5/bacula-restore
        state: directory
        owner: bacula
        group: bacula
        mode: '0750'

    - name: Configurar Bacula Director
      template:
        src: bacula-dir.conf.j2
        dest: /etc/bacula/bacula-dir.conf
        mode: '0640'
        owner: bacula
        group: bacula
      notify: restart bacula-director

   - name: Configurar Bacula Storage Daemon
      template:
        src: bacula-sd.conf.j2
        dest: /etc/bacula/bacula-sd.conf
        mode: '0640'
        owner: bacula
        group: bacula
      notify: restart bacula-sd

    - name: Habilitar y arrancar Bacula Director
      systemd:
        name: bacula-director
        state: started
        enabled: yes

    - name: Habilitar y arrancar Bacula Storage Daemon
      systemd:
        name: bacula-sd
        state: started
        enabled: yes

  handlers:
    - name: restart bacula-director
      systemd:
        name: bacula-director
        state: restarted

    - name: restart bacula-sd
      systemd:
        name: bacula-sd
        state: restarted

# INSTALACIÓN Y CONFIGURACIÓN DE RCLONE
- name: Instalar y configurar Rclone para Google Drive
  hosts: backup_servers
  become: yes
  vars:
    rclone_version: "current"
    
  tasks:
    - name: Instalar Rclone
      apt:
        name: rclone
        state: present
        update_cache: yes

    - name: Crear directorio de configuración Rclone
      file:
        path: /root/.config/rclone
        state: directory
        mode: '0700'
        owner: root
        group: root

    - name: Crear script de sincronización a Google Drive
      copy:
        dest: /usr/local/bin/sync-to-gdrive.sh
        mode: '0750'
        owner: root
        group: root
        content: |
                 #!/bin/bash
          
          # Script de sincronización de backups a Google Drive
          LOG_FILE="/var/log/rclone-sync.log"
          BACKUP_DIR="/raid5/bacula-backups"
          REMOTE="gdrive:backups/bacula"
          
          echo "=== Inicio de sincronización: $(date) ===" >> "$LOG_FILE"
          
          # Verificar que rclone está configurado
          if ! rclone listremotes | grep -q "gdrive:"; then
              echo "ERROR: Rclone no configurado" >> "$LOG_FILE"
              exit 1
          fi
          
          # Sincronizar backups
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
              echo "ERROR en la sincronización: $(date)" >> "$LOG_FILE"
              exit 1
          fi
          
          # Limpiar backups antiguos (>30 días)
          find "$BACKUP_DIR" -type f -mtime +30 -delete
          
          echo "=== Fin de sincronización  ===" >> "$LOG_FILE"


    - name: Crear directorio para logs de Rclone
      file:
        path: /var/log
        state: directory
        mode: '0755'

    - name: Crear cron job para sincronización diaria
      cron:
        name: "Sincronización diaria a Google Drive"
        minute: "0"
        hour: "3"
        job: "/usr/local/bin/sync-to-gdrive.sh"
        user: root

    - name: Crear script de verificación de integridad
      copy:
        dest: /usr/local/bin/verify-backups.sh
        mode: '0750'
        owner: root
        group: root
        content: |
          #!/bin/bash

            LOG_FILE="/var/log/backup-verification.log"

            echo "=== Verificación: $(date) ===" >> "$LOG_FILE"

            # Verificar espacio RAID
            DISK_USAGE=$(df -h /raid5 | tail -1 | awk '{print $5}' | sed 's/%//')
            echo "Uso RAID5: ${DISK_USAGE}%" >> "$LOG_FILE"

            if [ "$DISK_USAGE" -gt 85 ]; then
              echo "ADVERTENCIA: Disco al ${DISK_USAGE}% - revisar limpieza de volúmenes" >> "$LOG_FILE"
            fi

            # Contar volúmenes de Bacula
            BACKUP_COUNT=$(find /raid5/bacula-backups -type f | wc -l)
            echo "Volúmenes en almacenamiento: $BACKUP_COUNT" >> "$LOG_FILE"

            # Verificar estado RAID - buscar discos fallidos (símbolo _)
            if grep -q "_" /proc/mdstat; then
              echo "CRÍTICO: Disco fallido detectado en RAID5 - intervención inmediata requerida" >> "$LOG_FILE"
            else
              echo "RAID5: OK - todos los discos operativos" >> "$LOG_FILE"
            fi

    - name: Crear cron job para verificación semanal
      cron:
        name: "Verificación semanal de backups"
        minute: "0"
        hour: "4"
        weekday: "1"
        job: "/usr/local/bin/verify-backups.sh"

    - name: Crear directorio para logs de Rclone
      file:
        path: /var/log
        state: directory
        mode: '0755'

    - name: Crear cron job para sincronización diaria
      cron:
        name: "Sincronización diaria a Google Drive"
        minute: "0"
        hour: "3"
        job: "/usr/local/bin/sync-to-gdrive.sh"
        user: root

    - name: Crear script de verificación de integridad
      copy:
        dest: /usr/local/bin/verify-backups.sh
        mode: '0750'
        owner: root
        group: root
        user: root

    - name: Crear script de restauración de emergencia
      copy:
        dest: /usr/local/bin/emergency-restore.sh
        mode: '0750'
        owner: root
        group: root
        content: |
         #!/bin/bash

            if [ "$#" -ne 2 ]; then
              echo "Uso: $0 <nombre_servidor> <ruta_destino>"
              echo "Ejemplo: $0 dc01 /raid5/bacula-restore"
              exit 1
            fi

            SERVIDOR=$1
            DESTINO=$2

            echo "Restaurando backups de $SERVIDOR a $DESTINO"

            # Descarga solo la carpeta del servidor indicado
            rclone copy "gdrive:backups/bacula/$SERVIDOR" "$DESTINO" \
              --progress \
              --transfers 4

            if [ $? -eq 0 ]; then
              echo "Descarga completada. Use bconsole para restaurar los archivos."
            else
              echo "ERROR durante la descarga desde Google Drive."
              exit 1
            fi

```
<img width="1335" height="831" alt="ansible_bacula2_yml" src="https://github.com/user-attachments/assets/0de510be-0a34-4ee4-a3c8-ce3c34a17733" />

---

## 6.6 Seguridad de Ansible

### 6.6.1 Protección de Archivos

```bash
# Permisos restrictivos
chmod 600 /home/ansible/hosts.ini
chmod 600 /home/ansible/*.yml
chmod 400 /home/ansible/.ssh/id_rsa_ansible
```
### 6.6.2 Ansible Vault

Para proteger información sensible se cifran con **Ansible Vault** (AES-256) 
dos archivos con propósitos distintos:

- `hosts.ini` — contiene IPs y estructura de red
- `secrets.yml` — contiene contraseñas y credenciales

**Cifrar ambos archivos con una contraseña maestra:**
```bash
ansible-vault encrypt /home/ansible/hosts.ini
ansible-vault encrypt /home/ansible/secrets.yml
```

**Ejecutar playbooks con ambos cifrados:**
```bash
ansible-playbook -i hosts.ini update.yml -e @secrets.yml --ask-vault-pass --ask-become-pass
```

**Editar sin descifrar permanentemente:**
```bash
ansible-vault edit /home/ansible/hosts.ini
ansible-vault edit /home/ansible/secrets.yml
```

**Descifrar si fuera necesario:**
```bash
ansible-vault decrypt /home/ansible/hosts.ini
ansible-vault decrypt /home/ansible/secrets.yml
```
### 6.6.3 Auditoría

Todas las ejecuciones de Ansible quedan registradas en los logs del sistema y son monitorizadas por auditd.

---

## 6.7 Comandos Útiles

| Comando | Descripción |
|---------|-------------|
| `ansible all -m ping` | Verificar conectividad |
| `ansible all -m shell -a "uptime"` | Ejecutar comando en todos |
| `ansible-playbook -C playbook.yml` | Dry-run (sin aplicar cambios) |
| `ansible-playbook --limit dc01` | Ejecutar solo en un host |
| `ansible-inventory --list` | Ver inventario en JSON |

---

## 6.8 Resultados

La implementación de Ansible ha permitido:

- ✅ Aplicar hardening consistente en todos los servidores
- ✅ Reducir tiempo de configuración de horas a minutos
- ✅ Garantizar configuraciones homogéneas
- ✅ Manipular la infraestructura como código (IaC)

# 06 - Automatización con Ansible

## 6.1 Introducción

Este documento describe la implementación de Ansible como herramienta de automatización para la gestión centralizada de configuraciones en la infraestructura TechLogix.

Ansible permite ejecutar tareas de forma consistente en múltiples servidores simultáneamente, garantizando configuraciones homogéneas y reduciendo errores humanos.

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

### 6.3.3 Distribuir Clave SSH

```bash
# Copiar clave a cada servidor (ejecutar para cada uno)
sudo -u ansible ssh-copy-id -i /home/ansible/.ssh/id_rsa_ansible.pub -p 2222 sysadmin@192.168.40.10
sudo -u ansible ssh-copy-id -i /home/ansible/.ssh/id_rsa_ansible.pub -p 2222 sysadmin@192.168.40.14
sudo -u ansible ssh-copy-id -i /home/ansible/.ssh/id_rsa_ansible.pub -p 2222 sysadmin@192.168.40.12
sudo -u ansible ssh-copy-id -i /home/ansible/.ssh/id_rsa_ansible.pub -p 2222 sysadmin@192.168.40.13
sudo -u ansible ssh-copy-id -i /home/ansible/.ssh/id_rsa_ansible.pub -p 2222 sysadmin@192.168.100.8
sudo -u ansible ssh-copy-id -i /home/ansible/.ssh/id_rsa_ansible.pub -p 2222 sysadmin@192.168.60.10
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
ansible all -i hosts.ini -m ping --ask-become-pass
```

Resultado esperado:
```
dc01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
dc02 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
...
```

---

## 6.5 Playbooks Implementados

### 6.5.1 Playbook de Securización (securizacion.yml)

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

**Archivo: securizacion.yml**

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

    - name: Configurar SSH - Deshabilitar autenticación por contraseña
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'

    - name: Configurar SSH - Cambiar puerto
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?Port'
        line: "Port {{ ssh_port }}"

    - name: Configurar SSH - MaxAuthTries
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?MaxAuthTries'
        line: 'MaxAuthTries 3'

    - name: Configurar SSH - Deshabilitar X11 Forwarding
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?X11Forwarding'
        line: 'X11Forwarding no'
      notify: restart ssh

    # 4. PERMISOS DE ARCHIVOS CRÍTICOS
    - name: Configurar permisos /etc/shadow
      file:
        path: /etc/shadow
        mode: '0600'
        owner: root
        group: root

    - name: Configurar permisos /etc/passwd
      file:
        path: /etc/passwd
        mode: '0644'
        owner: root
        group: root

    # 5. POLÍTICAS DE CONTRASEÑAS (PAM)
    - name: Configurar calidad de contraseñas - minlen
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?\s*minlen'
        line: 'minlen = 12'

    - name: Configurar calidad de contraseñas - minclass
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?\s*minclass'
        line: 'minclass = 3'

    # 6. CONFIGURAR AUDITD
    - name: Crear reglas de auditd
      blockinfile:
        path: /etc/audit/rules.d/hardening.rules
        create: yes
        block: |
          -w /etc/passwd -p wa -k passwd_changes
          -w /etc/group -p wa -k group_changes
          -w /etc/shadow -p wa -k shadow_changes
          -w /etc/sudoers -p wa -k sudoers_changes
      notify: restart auditd

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
      notify: restart fail2ban

    # 8. ACTUALIZACIONES AUTOMÁTICAS
    - name: Habilitar actualizaciones automáticas
      copy:
        dest: /etc/apt/apt.conf.d/20auto-upgrades
        content: |
          APT::Periodic::Update-Package-Lists "1";
          APT::Periodic::Unattended-Upgrade "1";
          APT::Periodic::AutocleanInterval "7";

    # 9. BANNER LEGAL
    - name: Crear banner legal de acceso
      copy:
        dest: /etc/motd
        content: |
          ****************************************************************
          ADVERTENCIA: Sistema de uso exclusivo para personal autorizado.
          Todas las actividades son monitorizadas y registradas.
          ****************************************************************

    # 10. DESHABILITAR SERVICIOS INNECESARIOS
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

    - name: restart auditd
      systemd:
        name: auditd
        state: restarted

    - name: restart fail2ban
      systemd:
        name: fail2ban
        state: restarted
```

**Ejecución:**

```bash
ansible-playbook -i hosts.ini securizacion.yml --ask-become-pass
```

---

### 6.5.2 Playbook UFW Granular (ufw_granular.yml)

Este playbook configura el firewall UFW de forma específica según el rol de cada servidor.

**Archivo: ufw_granular.yml**

```yaml
---
- name: Configuración UFW Base
  hosts: servers
  become: yes
  vars:
    ssh_port: 2222
    
  tasks:
    - name: Resetear UFW
      ufw:
        state: reset

    - name: Política por defecto - Denegar entrante
      ufw:
        direction: incoming
        policy: deny

    - name: Política por defecto - Permitir saliente
      ufw:
        direction: outgoing
        policy: allow

    - name: Permitir SSH
      ufw:
        rule: allow
        port: "{{ ssh_port }}"
        proto: tcp

    - name: Permitir ping
      ufw:
        rule: allow
        proto: icmp

    - name: Habilitar UFW
      ufw:
        state: enabled

# CONTROLADORES DE DOMINIO
- name: UFW para Domain Controllers
  hosts: domain_controllers
  become: yes
  tasks:
    - name: Permitir DNS
      ufw:
        rule: allow
        port: '53'

    - name: Permitir Kerberos
      ufw:
        rule: allow
        port: '88'

    - name: Permitir LDAP
      ufw:
        rule: allow
        port: '389'
        proto: tcp

    - name: Permitir LDAPS
      ufw:
        rule: allow
        port: '636'
        proto: tcp

    - name: Permitir SMB
      ufw:
        rule: allow
        port: '445'
        proto: tcp

    - name: Permitir Zabbix Agent
      ufw:
        rule: allow
        port: '10050'
        proto: tcp
        from_ip: 192.168.70.10

    - name: Permitir Bacula
      ufw:
        rule: allow
        port: '9101:9104'
        proto: tcp

# SERVIDOR DE ARCHIVOS
- name: UFW para File Server
  hosts: file_servers
  become: yes
  tasks:
    - name: Permitir SMB desde Admin
      ufw:
        rule: allow
        port: '445'
        proto: tcp
        from_ip: 192.168.10.0/24

    - name: Permitir FTP
      ufw:
        rule: allow
        port: '21'
        proto: tcp

    - name: Permitir FTP Pasivo
      ufw:
        rule: allow
        port: '40000:50000'
        proto: tcp

    - name: Permitir Zabbix Agent
      ufw:
        rule: allow
        port: '10050'
        proto: tcp
        from_ip: 192.168.70.10

# SERVIDOR WEB
- name: UFW para Web Server
  hosts: web_servers
  become: yes
  tasks:
    - name: Permitir HTTP
      ufw:
        rule: allow
        port: '80'
        proto: tcp

    - name: Permitir HTTPS
      ufw:
        rule: allow
        port: '443'
        proto: tcp

    - name: Permitir Zabbix Agent
      ufw:
        rule: allow
        port: '10050'
        proto: tcp
        from_ip: 192.168.70.10

    - name: Permitir Bacula FD
      ufw:
        rule: allow
        port: '9102'
        proto: tcp
        from_ip: 192.168.40.13

# SERVIDOR VPN
- name: UFW para Security Server
  hosts: security_servers
  become: yes
  tasks:
    - name: Permitir OpenVPN
      ufw:
        rule: allow
        port: '1194'
        proto: udp

    - name: Permitir tráfico VPN
      ufw:
        rule: allow
        direction: in
        interface: tun0

    - name: Permitir Zabbix Agent
      ufw:
        rule: allow
        port: '10050'
        proto: tcp
        from_ip: 192.168.70.10
```

**Ejecución:**

```bash
ansible-playbook -i hosts.ini ufw_granular.yml --ask-become-pass
```

---

### 6.5.3 Playbook de Backup (bacula_rclone.yml)

Este playbook configura los agentes de Bacula en todos los servidores.

**Archivo: bacula_rclone.yml**

```yaml
---
- name: Configurar Bacula File Daemon
  hosts: servers
  become: yes
  vars:
    bacula_director: 192.168.40.13
    bacula_password: "password_seguro"
    
  tasks:
    - name: Instalar Bacula File Daemon
      apt:
        name: bacula-fd
        state: present
        update_cache: yes

    - name: Configurar Bacula FD
      template:
        dest: /etc/bacula/bacula-fd.conf
        content: |
          Director {
            Name = bacula-dir
            Password = "{{ bacula_password }}"
          }
          
          FileDaemon {
            Name = {{ inventory_hostname }}-fd
            FDport = 9102
            WorkingDirectory = /var/lib/bacula
            Pid Directory = /run/bacula
            Maximum Concurrent Jobs = 20
          }
          
          Messages {
            Name = Standard
            director = bacula-dir = all, !skipped, !restored
          }
      notify: restart bacula-fd

    - name: Habilitar Bacula FD
      systemd:
        name: bacula-fd
        enabled: yes
        state: started

  handlers:
    - name: restart bacula-fd
      systemd:
        name: bacula-fd
        state: restarted
```

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

Para proteger contraseñas y datos sensibles:

```bash
# Cifrar archivo con credenciales
ansible-vault encrypt secrets.yml

# Ejecutar playbook con vault
ansible-playbook -i hosts.ini playbook.yml --ask-vault-pass --ask-become-pass
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
- ✅ Documentar la infraestructura como código
- ✅ Facilitar la recuperación ante desastres

---

## 6.9 Capturas del Proyecto

*[Añadir capturas de pantalla de:]*
- Ejecución de ansible ping a todos los servidores
- Resultado de playbook de securización
- Estado de fail2ban después de aplicar playbook

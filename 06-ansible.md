# Automatización con Ansible — InnovateTech

## Visión general

**Ansible** es una herramienta de automatización que permite desplegar, configurar y mantener infraestructuras de manera rápida y repetible sin necesidad de instalar agentes en las máquinas destino.

En este proyecto, Ansible automatiza la configuración de **7 instancias EC2** en AWS, creando usuarios administradores, configurando logs centralizados, instalando servicios (Icecast2, NGINX, Jellyfin, MariaDB, Jitsi, OpenLDAP) y sincronizando los ajustes entre todas las máquinas.

---

## Arquitectura

### Controlador Ansible

- **Máquina:** EC2 `ip-10-0-1-201` (Servidor de logs centralizados)
- **Ubicación de los archivos:** `/home/ubuntu/`
- **Clave privada SSH:** `/home/ubuntu/.ssh/labsuser.pem`

---

## Instalación de Ansible

### 1. Instalar Ansible en el controlador

```bash
# Actualizar repositorios
sudo apt update

# Instalar Ansible
sudo apt install ansible -y
```

### 2. Preparar la clave privada SSH

```bash
# Copiar la clave .pem a la máquina controladora (desde tu ordenador)
scp -i labsuser.pem labsuser.pem ubuntu@10.0.1.201:/home/ubuntu/.ssh/

# Conectar y establecer permisos
ssh -i labsuser.pem ubuntu@10.0.1.201

# En la máquina controladora:
chmod 400 /home/ubuntu/.ssh/labsuser.pem
```

---

## Configuración de Ansible

### 1. Archivo de inventario (`/etc/ansible/hosts`)

El inventario define las máquinas que Ansible gestiona y las variables de conexión:

```bash
sudo nano /etc/ansible/hosts
```

Añadir el siguiente contenido:

```ini
[servidors]
audio     ansible_host=10.0.1.60    ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/labsuser.pem
video     ansible_host=10.0.1.23    ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/labsuser.pem
bd        ansible_host=10.0.1.208   ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/labsuser.pem
jitsi     ansible_host=10.0.1.17    ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/labsuser.pem
ldap      ansible_host=10.0.1.209   ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/labsuser.pem
web       ansible_host=10.0.1.237   ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/labsuser.pem
logs      ansible_host=10.0.1.201   ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/labsuser.pem
```

**Explicación:**
- `[servidors]` — grupo de hosts
- `ansible_host` — IP privada de la máquina
- `ansible_user` — usuario SSH (ubuntu por defecto)
- `ansible_ssh_private_key_file` — ruta a la clave privada

### 2. Archivo de configuración (`/etc/ansible/ansible.cfg`)

```bash
sudo nano /etc/ansible/ansible.cfg
```

Añadir:

```ini
[defaults]
host_key_checking = False
allow_world_readable_tmpfiles = True

[privilege_escalation]
become_method = sudo
```

**Explicación:**
- `host_key_checking = False` — no pedir confirmación de fingerprint SSH
- `allow_world_readable_tmpfiles = True` — permite usar `become_user` sin errores de permisos
- `become_method = sudo` — usar `sudo` para escalada de privilegios

### 3. Verificar conectividad

```bash
# Probar conexión a todas las máquinas
ansible servidors -m ping
```

**Resultado esperado:**
```
audio | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
video | SUCCESS => { ... }
bd | SUCCESS => { ... }
...
```

---

## Playbooks

Los playbooks son archivos YAML que definen las tareas a ejecutar en las máquinas remotas.

### 1. `playbook_admin.yml` — Crear usuario administrador

Crea el usuario `adminitb` con permisos `sudo` y clave SSH en todas las máquinas.

```yaml
---
- name: Crear usuario admin específico
  hosts: servidors
  become: yes

  tasks:
    - name: Crear usuario adminitb
      user:
        name: adminitb
        shell: /bin/bash
        create_home: yes
        state: present

    - name: Añadir adminitb al grupo sudo
      user:
        name: adminitb
        groups: sudo
        append: yes

    - name: Crear directorio .ssh
      file:
        path: /home/adminitb/.ssh
        state: directory
        owner: adminitb
        group: adminitb
        mode: '0700'

    - name: Copiar clave pública SSH
      authorized_key:
        user: adminitb
        state: present
        key: "{{ lookup('file', '/home/ubuntu/.ssh/authorized_keys') }}"
```

**Tareas que realiza:**
1. Crea el usuario `adminitb` con shell `/bin/bash` y directorio home
2. Lo añade al grupo `sudo` (permisos de administrador)
3. Crea el directorio `.ssh` con permisos `700` (solo el usuario puede acceder)
4. Copia la clave SSH pública del controlador al usuario remoto (acceso sin contraseña)

---

### 2. `playbook_rsyslog.yml` — Configurar logs centralizados

Instala rsyslog (si no está) y configura todos los clientes para enviar logs al servidor central.

```yaml
---
- name: Configurar rsyslog agent en todos los servidores
  hosts: servidors
  become: yes

  tasks:
    - name: Instalar rsyslog
      apt:
        name: rsyslog
        state: present
        update_cache: yes

    - name: Añadir configuración para enviar logs al servidor central
      lineinfile:
        path: /etc/rsyslog.conf
        line: "*.* @10.0.1.201:514"
        state: present

    - name: Reiniciar rsyslog
      service:
        name: rsyslog
        state: restarted
        enabled: yes
```

**Tareas que realiza:**
1. Instala el paquete `rsyslog` (actualizando caché de `apt` si es necesario)
2. Añade la línea `*.* @10.0.1.201:514` al archivo `/etc/rsyslog.conf` (envía todos los logs al servidor `10.0.1.201` por UDP puerto 514)
3. Reinicia el servicio rsyslog y lo habilita para futuros arranques

---

### 3. `playbook_audio.yml` — Configurar servidor de streaming de audio

Instala y configura **Icecast2 + FFmpeg** para streaming de música.

```yaml
---
- name: Configurar servidor de audio Icecast2
  hosts: audio
  become: yes

  tasks:
    - name: Actualizar repositorios
      apt:
        update_cache: yes

    - name: Instalar Icecast2 y FFmpeg
      apt:
        name:
          - icecast2
          - ffmpeg
          - screen
        state: present

    - name: Configurar icecast.xml
      copy:
        dest: /etc/icecast2/icecast.xml
        content: |
          <icecast>
            <location>InnovateTech</location>
            <admin>admin@innovatetech.local</admin>
            <limits>
              <clients>100</clients>
              <sources>10</sources>
              <burst-on-connect>1</burst-on-connect>
            </limits>
            <authentication>
              <source-password>pirineus</source-password>
              <relay-password>pirineus</relay-password>
              <admin-user>admin</admin-user>
              <admin-password>pirineus</admin-password>
            </authentication>
            <hostname>32.194.168.28</hostname>
            <listen-socket>
              <port>8000</port>
            </listen-socket>
            <http-headers>
              <header name="Access-Control-Allow-Origin" value="*" />
            </http-headers>
            <fileserve>1</fileserve>
            <paths>
              <basedir>/usr/share/icecast2</basedir>
              <logdir>/var/log/icecast2</logdir>
              <webroot>/usr/share/icecast2/web</webroot>
              <adminroot>/usr/share/icecast2/admin</adminroot>
              <alias source="/" destination="/status.xsl"/>
            </paths>
            <logging>
              <accesslog>access.log</accesslog>
              <errorlog>error.log</errorlog>
              <loglevel>3</loglevel>
            </logging>
            <security>
              <chroot>0</chroot>
            </security>
          </icecast>

    - name: Habilitar y arrancar Icecast2
      service:
        name: icecast2
        state: started
        enabled: yes

    - name: Verificar que Icecast2 está activo
      service:
        name: icecast2
        state: started
```

**Tareas que realiza:**
1. Instala `icecast2`, `ffmpeg` y `screen`
2. Crea el archivo de configuración `/etc/icecast2/icecast.xml` con los parámetros:
   - Contraseña: `pirineus`
   - Puerto: `8000`
   - IP pública: `32.194.168.28` (actualizar según la IP real)
   - Límites: máximo 100 clientes, 10 fuentes
3. Inicia el servicio Icecast2 y lo habilita al arranque

---

### 4. `playbook_video.yml` — Configurar servidor NGINX RTMP + Jellyfin

```yaml
---
- name: Configurar servidor de vídeo NGINX RTMP + Jellyfin
  hosts: video
  become: yes

  tasks:
    - name: Actualizar repositorios
      apt:
        update_cache: yes

    - name: Instalar NGINX RTMP y FFmpeg
      apt:
        name:
          - nginx
          - libnginx-mod-rtmp
          - ffmpeg
          - screen
        state: present

    - name: Configurar nginx.conf
      copy:
        dest: /etc/nginx/nginx.conf
        content: |
          user www-data;
          worker_processes auto;
          pid /run/nginx.pid;
          include /etc/nginx/modules-enabled/*.conf;

          events {
              worker_connections 1024;
          }

          http {
              server {
                  listen 8080;
                  location /hls {
                      types {
                          application/vnd.apple.mpegurl m3u8;
                          video/mp2t ts;
                      }
                      root /tmp;
                      add_header Cache-Control no-cache;
                      add_header Access-Control-Allow-Origin *;
                  }
                  location / {
                      root /var/www/html;
                      index player.html;
                  }
              }
          }

          rtmp {
              server {
                  listen 1935;
                  chunk_size 4096;
                  application live {
                      live on;
                      record off;
                      hls on;
                      hls_path /tmp/hls;
                      hls_fragment 3;
                      hls_playlist_length 60;
                  }
              }
          }

    - name: Crear directorio HLS
      file:
        path: /tmp/hls
        state: directory
        mode: '0755'

    - name: Crear directorio web
      file:
        path: /var/www/html
        state: directory

    - name: Habilitar y arrancar NGINX
      service:
        name: nginx
        state: restarted
        enabled: yes

    - name: Instalar Jellyfin
      shell: curl -fsSL https://repo.jellyfin.org/install-debuntu.sh | bash
      args:
        creates: /usr/bin/jellyfin

    - name: Habilitar y arrancar Jellyfin
      service:
        name: jellyfin
        state: started
        enabled: yes
```

**Tareas que realiza:**
1. Instala NGINX con módulo RTMP, FFmpeg y Screen
2. Configura NGINX para:
   - Escuchar RTMP en el puerto 1935 (ingesta de streams)
   - Generar HLS en el puerto 8080 (reproducción en navegador)
3. Crea directorios `/tmp/hls` (fragmentos HLS) y `/var/www/html` (reproductor web)
4. Instala Jellyfin mediante el script oficial
5. Inicia ambos servicios (NGINX y Jellyfin)

---

### 5. `playbook_bd.yml` — Instalar MariaDB

```yaml
---
- name: Configurar servidor de base de datos MariaDB
  hosts: bd
  become: yes

  tasks:
    - name: Actualizar repositorios
      apt:
        update_cache: yes

    - name: Instalar MariaDB
      apt:
        name:
          - mariadb-server
          - python3-pymysql
        state: present

    - name: Habilitar y arrancar MariaDB
      service:
        name: mariadb
        state: started
        enabled: yes

    - name: Verificar que MariaDB está activo
      service:
        name: mariadb
        state: started
```

**Tareas que realiza:**
1. Instala `mariadb-server` (SGBD) y `python3-pymysql` (conector Python para Ansible)
2. Inicia el servicio MariaDB y lo habilita al arranque

---

### 6. `playbook_logs.yml` — Configurar servidor central de logs

```yaml
---
- name: Configurar servidor de logs centralizados
  hosts: logs
  become: yes

  tasks:
    - name: Instalar rsyslog
      apt:
        name: rsyslog
        state: present
        update_cache: yes

    - name: Crear directorio de logs remotos
      file:
        path: /var/log/remote
        state: directory
        owner: syslog
        group: syslog
        mode: '0755'

    - name: Configurar rsyslog como servidor central
      blockinfile:
        path: /etc/rsyslog.conf
        block: |
          module(load="imudp")
          input(type="imudp" port="514")
          module(load="imtcp")
          input(type="imtcp" port="514")
          $template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
          *.* ?RemoteLogs

    - name: Reiniciar rsyslog
      service:
        name: rsyslog
        state: restarted
        enabled: yes
```

**Tareas que realiza:**
1. Instala rsyslog (si no está)
2. Crea el directorio `/var/log/remote` con propietario `syslog` y permisos `755`
3. Configura rsyslog para escuchar UDP y TCP en el puerto 514
4. Define una plantilla que organiza los logs por hostname y programa: `/var/log/remote/<HOSTNAME>/<PROGRAMA>.log`
5. Reinicia rsyslog para aplicar los cambios

---

### 7. `playbook_jitsi.yml` — Instalar Jitsi Meet

```yaml
---
- name: Configurar servidor Jitsi Meet
  hosts: jitsi
  become: yes

  tasks:
    - name: Actualizar repositorios
      apt:
        update_cache: yes

    - name: Instalar Docker y Docker Compose
      apt:
        name:
          - docker.io
          - docker-compose
        state: present

    - name: Habilitar y arrancar Docker
      service:
        name: docker
        state: started
        enabled: yes

    - name: Clonar repositorio Jitsi Meet
      git:
        repo: https://github.com/jitsi/docker-jitsi-meet.git
        dest: /home/ubuntu/docker-jitsi-meet
        force: no

    - name: Verificar que Docker está activo
      service:
        name: docker
        state: started
```

**Tareas que realiza:**
1. Instala Docker y Docker Compose
2. Inicia el servicio Docker
3. Clona el repositorio oficial de Jitsi Meet (rama `master`)

---

### 8. `playbook_ldap_sftp.yml` — OpenLDAP + SFTP

```yaml
---
- name: Configurar servidor LDAP y SFTP
  hosts: ldap
  become: yes

  tasks:
    - name: Actualizar repositorios
      apt:
        update_cache: yes

    - name: Instalar OpenLDAP y utilidades
      apt:
        name:
          - slapd
          - ldap-utils
          - openssh-server
        state: present

    - name: Habilitar y arrancar slapd
      service:
        name: slapd
        state: started
        enabled: yes

    - name: Habilitar y arrancar SSH para SFTP
      service:
        name: ssh
        state: started
        enabled: yes

    - name: Crear grupo sftp
      group:
        name: sftpusers
        state: present

    - name: Configurar SFTP en sshd_config
      blockinfile:
        path: /etc/ssh/sshd_config
        block: |
          Match Group sftpusers
              ChrootDirectory /home/%u
              ForceCommand internal-sftp
              AllowTcpForwarding no

    - name: Reiniciar SSH
      service:
        name: ssh
        state: restarted
```

**Tareas que realiza:**
1. Instala slapd (servidor LDAP), utilidades LDAP y SSH
2. Inicia los servicios slapd y SSH
3. Crea el grupo `sftpusers` para usuarios SFTP
4. Configura SSH para encerrar usuarios SFTP en sus directorios (`/home/%u`)
5. Deshabilita el reenvío de puertos para usuarios SFTP (seguridad)

---

### 9. `playbook_web.yml` — Servidor web

```yaml
---
- name: Configurar servidor web
  hosts: web
  become: yes

  tasks:
    - name: Actualizar repositorios
      apt:
        update_cache: yes

    - name: Instalar Apache y SSH
      apt:
        name:
          - apache2
          - openssh-server
        state: present

    - name: Habilitar y arrancar Apache
      service:
        name: apache2
        state: started
        enabled: yes

    - name: Habilitar y arrancar SSH
      service:
        name: ssh
        state: started
        enabled: yes

    - name: Verificar que Apache está activo
      service:
        name: apache2
        state: started
```

**Tareas que realiza:**
1. Instala Apache2 y OpenSSH
2. Inicia ambos servicios y los habilita al arranque

---

## Playbook unificado: `site.yml`

Para ejecutar todos los playbooks de manera secuencial con un solo comando:

```yaml
---
- import_playbook: playbook_admin.yml
- import_playbook: playbook_rsyslog.yml
- import_playbook: playbook_audio.yml
- import_playbook: playbook_video.yml
- import_playbook: playbook_bd.yml
- import_playbook: playbook_logs.yml
- import_playbook: playbook_jitsi.yml
- import_playbook: playbook_ldap_sftp.yml
- import_playbook: playbook_web.yml
```

---

## Ejecución de los Playbooks

### Ejecutar un playbook individual

```bash
# Crear usuario admin en todas las máquinas
ansible-playbook /home/ubuntu/playbook_admin.yml

# Configurar logs en todos los servidores
ansible-playbook /home/ubuntu/playbook_rsyslog.yml

# Instalar audio solo en la máquina 'audio'
ansible-playbook /home/ubuntu/playbook_audio.yml
```

### Ejecutar todos los playbooks a la vez

```bash
# Desplegar toda la infraestructura
ansible-playbook /home/ubuntu/site.yml
```

### Ejecutar con verbosidad (ver los detalles)

```bash
# Modo verbose (muestra variables, resultados...)
ansible-playbook /home/ubuntu/site.yml -v

# Modo extra verbose (muestra tareas internas)
ansible-playbook /home/ubuntu/site.yml -vv

# Modo debug (todos los detalles)
ansible-playbook /home/ubuntu/site.yml -vvv
```

### Ejecución parcial

```bash
# Ejecutar solo tareas que empiecen por "Instal"
ansible-playbook /home/ubuntu/site.yml --tags "install"

# Saltar tareas específicas
ansible-playbook /home/ubuntu/site.yml --skip-tags "restart"

# Ejecutar sobre un host específico
ansible-playbook /home/ubuntu/playbook_audio.yml -i localhost,
```

---

## Validación

### 1. Verificar usuario adminitb en todas las máquinas

```bash
ansible servidors -m command -a "id adminitb"
```

**Resultado esperado:**
```
audio | CHANGED | rc=0 >>
uid=1001(adminitb) gid=1001(adminitb) groups=1001(adminitb),27(sudo)

video | CHANGED | rc=0 >>
uid=1001(adminitb) gid=1001(adminitb) groups=1001(adminitb),27(sudo)
...
```

### 2. Verificar logs centralizados

```bash
# En el servidor de logs
ls /var/log/remote/

# Resultado: carpetas con nombres de las otras máquinas
ip-10-0-1-60 ip-10-0-1-23 ip-10-0-1-208 ...

# Verificar contenido de logs
cat /var/log/remote/ip-10-0-1-208/sshd.log
```

### 3. Verificar servicios activos

```bash
# En la máquina 'audio'
ansible audio -m service -a "name=icecast2 state=started"

# En la máquina 'video'
ansible video -m service -a "name=nginx state=started"

# En la máquina 'bd'
ansible bd -m service -a "name=mariadb state=started"
```

### 4. Test de conectividad entre máquinas

```bash
# Desde el controlador, verificar que la máquina 'bd' puede llegar a 'logs'
ansible bd -m command -a "ping -c 1 10.0.1.201"
```

---

## Estructura de archivos

```
/home/ubuntu/
├── playbook_admin.yml
├── playbook_rsyslog.yml
├── playbook_audio.yml
├── playbook_video.yml
├── playbook_bd.yml
├── playbook_logs.yml
├── playbook_jitsi.yml
├── playbook_ldap_sftp.yml
├── playbook_web.yml
├── site.yml
└── .ssh/
    └── labsuser.pem

/etc/ansible/
├── hosts              # Inventario
├── ansible.cfg        # Configuración global
└── roles/             # (opcional) Roles reutilizables
```

---

## Resolución de problemas

| Error | Causa | Solución |
|-------|-------|----------|
| **"Permission denied (publickey)"** | Clave SSH no accesible | Verificar que `/home/ubuntu/.ssh/labsuser.pem` existe y tiene permisos `400` |
| **"Connection refused"** | Puerto SSH bloqueado | Verificar Security Group: abrir puerto 22 TCP |
| **"UNREACHABLE"** | Máquina no responde | Verificar que la máquina está en ejecución en AWS |
| **"changed failed"** | Tarea Ansible falló | Ver el output completo con `-vv` para diagnosticar |
| **"missing required module"** | Módulo Python ausente | Instalar vía `apt` (ej: `python3-pymysql`) |
| **"lineinfile: file not found"** | Archivo no existe | Asegurarse de que el archivo existe o usar `create: yes` |

---

## Referencia rápida de comandos

| Comando | Función |
|---------|---------|
| `ansible servidors -m ping` | Probar conexión a todas las máquinas |
| `ansible servidors -m command -a "uptime"` | Ejecutar comando remoto |
| `ansible-playbook playbook.yml` | Ejecutar playbook |
| `ansible-playbook playbook.yml -v` | Modo verbose |
| `ansible-playbook playbook.yml --list-tasks` | Listar las tareas sin ejecutar |
| `ansible-playbook playbook.yml --list-hosts` | Listar las máquinas afectadas |
| `ansible servidors -b -m apt -a "name=vim state=present"` | Instalar paquete remoto con sudo |
| `ansible servidors -m gather_facts` | Recopilar información del sistema |

---

## Ventajas de Ansible en este proyecto

✅ **Repetibilidad:** Los mismos playbooks se pueden ejecutar múltiples veces sin efectos secundarios.

✅ **Escalabilidad:** ¿Añadir nueva máquina? Actualizar inventario y ejecutar `site.yml`.

✅ **Documentación viva:** Los playbooks son la documentación de la infraestructura.

✅ **Seguridad:** Sin agentes instalados, solo necesita clave SSH.

✅ **Rapidez:** Configurar 7 servidores en minutos en vez de horas manuales.

✅ **Versionable:** Los playbooks se guardan en Git para control de versiones.

---

## Conclusión

Ansible proporciona automatización sin agentes que permite desplegar y mantener la infraestructura de **Innovate Tech** de manera rápida, repetible y segura. Los playbooks definen la configuración deseada, garantizando consistencia entre todas las máquinas.

# Automatització amb Ansible — InnovateTech

## Visió general

**Ansible** és una eina d'automatització que permet desplegar, configurar i mantenir infraestructures de manera ràpida i repetible sense necessitat d'instal·lar agents en les màquines destí.

En aquest projecte, Ansible automatitza la configuració de **7 instàncies EC2** en AWS, creant usuaris administradors, configurant logs centralitzats, instal·lant serveis (Icecast2, NGINX, Jellyfin, MariaDB, Jitsi, OpenLDAP) i sincronitzant els ajustaments entre totes les màquines.

---

## Arquitectura

### Controlador Ansible

- **Màquina:** EC2 `ip-10-0-1-201` (Servidor de logs centralitzats)
- **Ubicació dels fitxers:** `/home/ubuntu/`
- **Clau privada SSH:** `/home/ubuntu/.ssh/labsuser.pem`


---

## Instal·lació d'Ansible

### 1. Instal·lar Ansible al controlador

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar Ansible
sudo apt install ansible -y




### 2. Preparar la clau privada SSH

```bash
# Copiar la clau .pem a la màquina controladora (desde el teu ordenador)
scp -i labsuser.pem labsuser.pem ubuntu@10.0.1.201:/home/ubuntu/.ssh/

# Conectar i establir permisos
ssh -i labsuser.pem ubuntu@10.0.1.201

# A la màquina controladora:
chmod 400 /home/ubuntu/.ssh/labsuser.pem
```

---

## Configuració d'Ansible

### 1. Fitxer d'inventari (`/etc/ansible/hosts`)

El inventari defineix les màquines que Ansible gestiona i variables de connexió:

```bash
sudo nano /etc/ansible/hosts
```

Afegir el següent contingut:

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

**Explicació:**
- `[servidors]` — grup de hosts
- `ansible_host` — IP privada de la màquina
- `ansible_user` — usuari SSH (ubuntu per defecte)
- `ansible_ssh_private_key_file` — ruta a la clau privada

### 2. Fitxer de configuració (`/etc/ansible/ansible.cfg`)

```bash
sudo nano /etc/ansible/ansible.cfg
```

Afegir:

```ini
[defaults]
host_key_checking = False
allow_world_readable_tmpfiles = True

[privilege_escalation]
become_method = sudo
```

**Explicació:**
- `host_key_checking = False` — no demanar confirmació de fingerprint SSH
- `allow_world_readable_tmpfiles = True` — permetreix usar `become_user` sense errors de permisos
- `become_method = sudo` — usar `sudo` per escalada de privilegis

### 3. Verificar conectivitat

```bash
# Provar connexió a totes les màquines
ansible servidors -m ping
```

**Resultat espertat:**
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

Els playbooks són fitxers YAML que defineixen les tasques a executar en les màquines remotes.

### 1. `playbook_admin.yml` — Crear usuari administrador

Crea l'usuari `adminitb` amb permisos `sudo` i clave SSH a totes les màquines.

```yaml
---
- name: Crear usuari admin específic
  hosts: servidors
  become: yes

  tasks:
    - name: Crear usuari adminitb
      user:
        name: adminitb
        shell: /bin/bash
        create_home: yes
        state: present

    - name: Afegir adminitb al grup sudo
      user:
        name: adminitb
        groups: sudo
        append: yes

    - name: Crear directori .ssh
      file:
        path: /home/adminitb/.ssh
        state: directory
        owner: adminitb
        group: adminitb
        mode: '0700'

    - name: Copiar clau pública SSH
      authorized_key:
        user: adminitb
        state: present
        key: "{{ lookup('file', '/home/ubuntu/.ssh/authorized_keys') }}"
```

**Tasques que fa:**
1. Crea l'usuari `adminitb` amb shell `/bin/bash` i home directory
2. L'afegeix al grup `sudo` (permisos d'administrador)
3. Crea el directori `.ssh` amb permisos `700` (només l'usuari pot accedir)
4. Copia la clau SSH pública del controlador al usuari remot (accés sense contrasenya)

---

### 2. `playbook_rsyslog.yml` — Configurar logs centralitzats

Instal·la rsyslog (si no està) i configura tots els clientes per enviar logs al servidor central.

```yaml
---
- name: Configurar rsyslog agent a tots els servidors
  hosts: servidors
  become: yes

  tasks:
    - name: Instal·lar rsyslog
      apt:
        name: rsyslog
        state: present
        update_cache: yes

    - name: Afegir configuració per enviar logs al servidor central
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

**Tasques que fa:**
1. Instal·la el paquet `rsyslog` (actualitzant cache de `apt` si és necessari)
2. Afegeix la línia `*.* @10.0.1.201:514` al fitxer `/etc/rsyslog.conf` (envia tots els logs al servidor `10.0.1.201` per UDP port 514)
3. Reinicia el servei rsyslog i l'habilita per a futur boot

---

### 3. `playbook_audio.yml` — Configurar servidor de streaming d'audio

Instal·la i configura **Icecast2 + FFmpeg** per a streaming de música.

```yaml
---
- name: Configurar servidor d'audio Icecast2
  hosts: audio
  become: yes

  tasks:
    - name: Actualitzar repositoris
      apt:
        update_cache: yes

    - name: Instal·lar Icecast2 i FFmpeg
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

    - name: Habilitar i arrancar Icecast2
      service:
        name: icecast2
        state: started
        enabled: yes

    - name: Verificar que Icecast2 està actiu
      service:
        name: icecast2
        state: started
```

**Tasques que fa:**
1. Instal·la `icecast2`, `ffmpeg` i `screen`
2. Crea el fitxer de configuració `/etc/icecast2/icecast.xml` amb els paràmetres:
   - Contrasenya: `pirineus`
   - Port: `8000`
   - IP pública: `32.194.168.28` (actualitzar segons la teva IP real)
   - Límits: màxim 100 clients, 10 fonts
3. Inicia el servei Icecast2 i l'habilita al boot

---

### 4. `playbook_video.yml` — Configurar servidor NGINX RTMP + Jellyfin

```yaml
---
- name: Configurar servidor de video NGINX RTMP + Jellyfin
  hosts: video
  become: yes

  tasks:
    - name: Actualitzar repositoris
      apt:
        update_cache: yes

    - name: Instal·lar NGINX RTMP i FFmpeg
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

    - name: Crear directori HLS
      file:
        path: /tmp/hls
        state: directory
        mode: '0755'

    - name: Crear directori web
      file:
        path: /var/www/html
        state: directory

    - name: Habilitar i arrancar NGINX
      service:
        name: nginx
        state: restarted
        enabled: yes

    - name: Instal·lar Jellyfin
      shell: curl -fsSL https://repo.jellyfin.org/install-debuntu.sh | bash
      args:
        creates: /usr/bin/jellyfin

    - name: Habilitar i arrancar Jellyfin
      service:
        name: jellyfin
        state: started
        enabled: yes
```

**Tasques que fa:**
1. Instal·la NGINX amb mòdul RTMP, FFmpeg i Screen
2. Configura NGINX per:
   - Escoltar RTMP al port 1935 (ingesta de streams)
   - Generar HLS al port 8080 (reproducció en navegador)
3. Crea directoris `/tmp/hls` (fragments HLS) i `/var/www/html` (reproductor web)
4. Instal·la Jellyfin via script oficial
5. Inicia ambdós serveis (NGINX i Jellyfin)

---

### 5. `playbook_bd.yml` — Instal·lar MariaDB

```yaml
---
- name: Configurar servidor de base de dades MariaDB
  hosts: bd
  become: yes

  tasks:
    - name: Actualitzar repositoris
      apt:
        update_cache: yes

    - name: Instal·lar MariaDB
      apt:
        name:
          - mariadb-server
          - python3-pymysql
        state: present

    - name: Habilitar i arrancar MariaDB
      service:
        name: mariadb
        state: started
        enabled: yes

    - name: Verificar que MariaDB està actiu
      service:
        name: mariadb
        state: started
```

**Tasques que fa:**
1. Instal·la `mariadb-server` (SGBD) i `python3-pymysql` (connector Python per Ansible)
2. Inicia el servei MariaDB i l'habilita al boot

---

### 6. `playbook_logs.yml` — Configurar servidor central de logs

```yaml
---
- name: Configurar servidor de logs centralitzats
  hosts: logs
  become: yes

  tasks:
    - name: Instal·lar rsyslog
      apt:
        name: rsyslog
        state: present
        update_cache: yes

    - name: Crear directori de logs remots
      file:
        path: /var/log/remote
        state: directory
        owner: syslog
        group: syslog
        mode: '0755'

    - name: Configurar rsyslog com a servidor central
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

**Tasques que fa:**
1. Instal·la rsyslog (si no està)
2. Crea el directori `/var/log/remote` amb propietari `syslog` i permisos `755`
3. Configura rsyslog per escoltar UDP i TCP al port 514
4. Defineix una plantilla que organitza logs per hostname i programa: `/var/log/remote/<HOSTNAME>/<PROGRAMA>.log`
5. Reinicia rsyslog per aplicar els canvis

---

### 7. `playbook_jitsi.yml` — Instal·lar Jitsi Meet

```yaml
---
- name: Configurar servidor Jitsi Meet
  hosts: jitsi
  become: yes

  tasks:
    - name: Actualitzar repositoris
      apt:
        update_cache: yes

    - name: Instal·lar Docker i Docker Compose
      apt:
        name:
          - docker.io
          - docker-compose
        state: present

    - name: Habilitar i arrancar Docker
      service:
        name: docker
        state: started
        enabled: yes

    - name: Clonar repositori Jitsi Meet
      git:
        repo: https://github.com/jitsi/docker-jitsi-meet.git
        dest: /home/ubuntu/docker-jitsi-meet
        force: no

    - name: Verificar que Docker està actiu
      service:
        name: docker
        state: started
```

**Tasques que fa:**
1. Instal·la Docker i Docker Compose
2. Inicia el servei Docker
3. Clona el repositori oficial de Jitsi Meet (branca `master`)

---

### 8. `playbook_ldap_sftp.yml` — OpenLDAP + SFTP

```yaml
---
- name: Configurar servidor LDAP i SFTP
  hosts: ldap
  become: yes

  tasks:
    - name: Actualitzar repositoris
      apt:
        update_cache: yes

    - name: Instal·lar OpenLDAP i utilitats
      apt:
        name:
          - slapd
          - ldap-utils
          - openssh-server
        state: present

    - name: Habilitar i arrancar slapd
      service:
        name: slapd
        state: started
        enabled: yes

    - name: Habilitar i arrancar SSH per SFTP
      service:
        name: ssh
        state: started
        enabled: yes

    - name: Crear grup sftp
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

**Tasques que fa:**
1. Instal·la slapd (servidor LDAP), utilitats LDAP i SSH
2. Inicia els serveis slapd i SSH
3. Crea el grup `sftpusers` per a usuaris SFTP
4. Configura SSH per encarcerar usuaris SFTP en els seus directoris (`/home/%u`)
5. Deshabilita forwarding de portes per a usuaris SFTP (seguretat)

---

### 9. `playbook_web.yml` — Servidor web

```yaml
---
- name: Configurar servidor web
  hosts: web
  become: yes

  tasks:
    - name: Actualitzar repositoris
      apt:
        update_cache: yes

    - name: Instal·lar Apache i SSH
      apt:
        name:
          - apache2
          - openssh-server
        state: present

    - name: Habilitar i arrancar Apache
      service:
        name: apache2
        state: started
        enabled: yes

    - name: Habilitar i arrancar SSH
      service:
        name: ssh
        state: started
        enabled: yes

    - name: Verificar que Apache està actiu
      service:
        name: apache2
        state: started
```

**Tasques que fa:**
1. Instal·la Apache2 i OpenSSH
2. Inicia ambdós serveis i els habilita al boot

---

## Playbook unificado: `site.yml`

Per executar tots els playbooks de manera seqüencial amb un sol comando:

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

## Execució dels Playbooks

### Executar un playbook individual

```bash
# Crear usuari admin a totes les màquines
ansible-playbook /home/ubuntu/playbook_admin.yml

# Configurar logs a tots els servidors
ansible-playbook /home/ubuntu/playbook_rsyslog.yml

# Instal·lar audio només a la màquina 'audio'
ansible-playbook /home/ubuntu/playbook_audio.yml
```

### Executar tots els playbooks de una vegada

```bash
# Desplegar tota la infraestructura
ansible-playbook /home/ubuntu/site.yml
```

### Executar amb verbositat (veure els detalles)

```bash
# Mode verbose (mostra variables, resultats...)
ansible-playbook /home/ubuntu/site.yml -v

# Mode extra verbose (mostra tasques internes)
ansible-playbook /home/ubuntu/site.yml -vv

# Mode debug (tots els detalls)
ansible-playbook /home/ubuntu/site.yml -vvv
```

### Execució parcial

```bash
# Executar només tasques que comencin amb "Instal"
ansible-playbook /home/ubuntu/site.yml --tags "install"

# Saltar tasques específiques
ansible-playbook /home/ubuntu/site.yml --skip-tags "restart"

# Executar sobre un host específic
ansible-playbook /home/ubuntu/playbook_audio.yml -i localhost,
```

---

## Validació

### 1. Verificar usuari adminitb en totes les màquines

```bash
ansible servidors -m command -a "id adminitb"
```

**Resultat espertat:**
```
audio | CHANGED | rc=0 >>
uid=1001(adminitb) gid=1001(adminitb) groups=1001(adminitb),27(sudo)

video | CHANGED | rc=0 >>
uid=1001(adminitb) gid=1001(adminitb) groups=1001(adminitb),27(sudo)
...
```

### 2. Verificar logs centralitzats

```bash
# Al servidor de logs
ls /var/log/remote/

# Resultat: carpetes amb noms de les altres màquines
ip-10-0-1-60 ip-10-0-1-23 ip-10-0-1-208 ...

# Verificar contingut de logs
cat /var/log/remote/ip-10-0-1-208/sshd.log
```

### 3. Verificar serveis actius

```bash
# En la màquina 'audio'
ansible audio -m service -a "name=icecast2 state=started"

# En la màquina 'video'
ansible video -m service -a "name=nginx state=started"

# En la màquina 'bd'
ansible bd -m service -a "name=mariadb state=started"
```

### 4. Test de connectivitat dins de les màquines

```bash
# Des del controlador, verificar que la màquina 'bd' pot arribar a 'logs'
ansible bd -m command -a "ping -c 1 10.0.1.201"
```

---

## Estructura de fitxers

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
├── hosts              # Inventari
├── ansible.cfg        # Configuració global
└── roles/             # (opcional) Rols reutilitzables
```

---

## Resolució de problemes

| Error | Causa | Solució |
|-------|-------|---------|
| **"Permission denied (publickey)"** | Clau SSH no accessible | Verificar que `/home/ubuntu/.ssh/labsuser.pem` existeix i té permisos `400` |
| **"Connection refused"** | Port SSH bloquejat | Verificar Security Group: obrir port 22 TCP |
| **"UNREACHABLE"** | Màquina no respón | Verificar que la màquina està en execució en AWS |
| **"changed failed"** | Tasca Ansible falló | Veure l'output complet amb `-vv` per diagnosticar |
| **"missing required module"** | Mòdul Python mancat | Instal·lar via `apt` (ex: `python3-pymysql`) |
| **"lineinfile: file not found"** | Fitxer no existeix | Assegurar que el fitxer existeix o usar `create: yes` |

---

## Referència ràpida de comandos

| Comanda | Funció |
|---------|--------|
| `ansible servidors -m ping` | Provar connexió a totes les màquines |
| `ansible servidors -m command -a "uptime"` | Executar comanda remota |
| `ansible-playbook playbook.yml` | Executar playbook |
| `ansible-playbook playbook.yml -v` | Verbose mode |
| `ansible-playbook playbook.yml --list-tasks` | Llistar les tasques sense executar |
| `ansible-playbook playbook.yml --list-hosts` | Llistar les màquines afectades |
| `ansible servidors -b -m apt -a "name=vim state=present"` | Instal·lar paquet remot amb sudo |
| `ansible servidors -m gather_facts` | Recopilar informació del sistema |

---

## Avantatges d'Ansible en aquest projecte

✅ **Repetibilitat:** Els mateixos playbooks es poden executar múltiples vegades sense efectes secundaris.

✅ **Escalabilitat:** Afegir nova màquina? Actualitzar inventari i executar `site.yml`.

✅ **Documentació viva:** Els playbooks són la documentació de la infraestructura.

✅ **Seguretat:** Sense agents instal·lats, només necessita clau SSH.

✅ **Rapidesa:** Configurar 7 servidors en minuts en comptes de hores manuals.

✅ **Versionable:** Els playbooks es guarden en Git per a control de versions.

---

## Conclusió

Ansible proporciona automatització sense agents que permet desplegar i mantenir la infraestructura de **Innovate Tech** de manera ràpida, repetible i segura. Els playbooks defineixen la configuració desitjada, garantint consistència entre totes les màquines.


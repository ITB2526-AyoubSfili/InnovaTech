# Proyecto Transversal ASIXc1 – Innovate Tech

**Título del proyecto:** pro-asixc1-grup  
**Miembros:** Ayoub, Didac, Faiver  
**Fecha de entrega:** 28 de mayo de 2026  
**Vídeo demostración:** [https://drive.google.com/file/d/1JMvHhW3vPhSh7vjXW4Uku7a2k6_LjNJS/view?usp=sharing]


['Informe completo proyecto'](Informe-completo-proyecto-ASIXc1-G6.pdf)
---

## Resumen 

Innovate Tech es una empresa tecnológica que necesita modernizar su infraestructura de comunicaciones y datos. Este proyecto consiste en diseñar e implantar un centro de procesamiento de datos (CPD) en la nube de AWS, junto con servicios multimedia (streaming de audio, vídeo y videoconferencia) y una base de datos corporativa completa.

Se ha utilizado **Ansible** para automatizar la configuración de los servidores, **rsyslog** para centralizar todos los logs, y **MariaDB** como motor de base de datos con triggers de auditoría, control de accesos por roles y copias de seguridad automáticas.

El resultado es una infraestructura **escalable, segura y documentada** que permite a Innovate Tech mejorar su comunicación interna y externa, distribuir contenidos digitales y gestionar su personal de forma eficiente.

## Estructura del repositorio

| Archivo | Contenido |
|---------|-----------|
| [`README.md`](README.md) | Este documento. Visión general del proyecto. |
| [`01-cpd-fisico.md`](01-cpd-fisico.md) | Diseño del CPD físico: ubicación, racks, climatización, SAI, seguridad, PRL. |
| [`02-arquitectura-aws.md`](02-arquitectura-aws.md) | Infraestructura en AWS: instancias EC2, VPC, grupos de seguridad, diagrama de red. |
| [`03-servidor-web-sftp.md`](03-servidor-web-sftp.md) | Servidor web (NGINX) y servidor SFTP con autenticación LDAP. |
| [`04-servidor-ldap.md`](04-servidor-ldap.md) | Directorio LDAP (OpenLDAP) para la gestión centralizada de usuarios. |
| [`05-centralizacion-logs.md`](05-centralizacion-logs.md) | Servidor central de logs con rsyslog. Configuración y comprobaciones. |
| [`06-ansible.md`](06-ansible.md) | Automatización con Ansible: playbooks, inventario, configuración de usuarios y servicios. |
| [`07-servicio-audio.md`](07-servicio-audio.md) | Servidor de streaming de audio (Icecast2). Instalación, configuración y pruebas. |
| [`08-servicio-video.md`](08-servicio-video.md) | Servidor de streaming de vídeo (NGINX RTMP + HLS) y Jellyfin. |
| [`09-videoconferencia.md`](09-videoconferencia.md) | Servidor de videoconferencia (Jitsi Meet con Docker). |
| [`10-amplada-banda.md`](10-amplada-banda.md) | Pruebas de ancho de banda, análisis y propuestas de mejora. |
| [`11-base-datos-diseno.md`](11-base-datos-diseno.md) | Diagrama Entidad-Relación y modelo relacional de la base de datos. |
| [`12-base-datos-implementacion.md`](12-base-datos-implementacion-triggers.md) | Scripts SQL de creación de tablas e inserción de datos de prueba. |
| [`13-triggers-auditoria.md`](13-triggers-auditoria.md) | Triggers de control de cuotas, bloqueo de usuarios y auditoría. |
| [`14-backups-eventos.md`](14-backups-eventos.md) | Copias de seguridad automáticas con eventos programados en MariaDB. |
| [`15-roles-usuarios-script.md`](15-roles-usuarios-script.md) | Roles (admin, vendes, administracio, treballador) y script Bash de creación de usuarios. |
| [`16-digitalizacion-seguridad.md`](16-digitalizacion-seguridad.md) | Transformación digital, seguridad física y lógica, importancia de los datos. |

---

## Resumen de la solución técnica

### Infraestructura en AWS (Módulo 0371 / 1665)

**7 instancias EC2** con Ubuntu 24.04 LTS, distribuidas en una VPC con subred privada (región us-east-1):

| Instancia | IP Privada | Servicio |
|-----------|-----------|---------|
| ec2-audio | 10.0.1.60 | Streaming de audio (Icecast2) |
| ec2-video | 10.0.1.23 | Streaming de vídeo (NGINX RTMP + Jellyfin) |
| ec2-bbdd | 10.0.1.208 | Base de datos (MariaDB) |
| ec2-logs | 10.0.1.201 | Logs centralizados (rsyslog + ansible) |
| ec2-jitsi | 10.0.1.17 | Videoconferencia (Jitsi Meet) |
| ec2-ldap | 10.0.1.209 | Directorio LDAP (OpenLDAP) + SFTP |
| ec2-web | 10.0.1.237 | Servidor web (NGINX) |

**Automatización con Ansible** desde el servidor de logs (`ec2-logs`):
- Inventario en `/etc/ansible/hosts` con grupo `servidors` y las 7 IPs privadas.
- `playbook_admin.yml` — crea el usuario `adminitb` con sudo y clave SSH en las 7 máquinas.
- `playbook_rsyslog.yml` — instala rsyslog y configura el envío de logs al servidor central en todas las máquinas.
- `playbook_audio.yml` — instala y configura Icecast2 + FFmpeg.
- `playbook_video.yml` — instala y configura NGINX RTMP + Jellyfin.
- `playbook_bd.yml` — instala y arranca MariaDB.
- `playbook_logs.yml` — configura el servidor central de logs.
- `playbook_jitsi.yml` — instala Docker y clona el repositorio de Jitsi Meet.
- `playbook_ldap_sftp.yml` — instala OpenLDAP y configura SFTP con chroot.
- `playbook_web.yml` — instala Apache y configura el servidor web.
- `site.yml` — agrupa todos los playbooks para despliegue completo con un solo comando.

**Seguridad:** acceso SSH exclusivamente por clave pública/privada, usuario administrador específico `adminitb` (no root), Security Groups con reglas mínimas por servicio.

---

### CPD físico — diseño teórico (Módulo 0371)

- **Ubicación:** Planta 2 interior, sin ventanas, suelo con resistencia ≥ 1.200 kg/m².
- **Climatización:** 2 unidades CRAC Liebert DS (N+1), temperatura 20–22 °C, humedad 45–55 %, filosofía pasillo frío/caliente, objetivo PUE < 1,4.
- **Cableado:** Cat6A U/FTP para datos, fibra OM4 para uplinks 10G, etiquetado alfanumérico.
- **Suelo técnico:** elevado 40 cm, placas de acero 60×60 cm perforadas en pasillo frío. Techo técnico a 3 m.
- **Racks:** 2 racks de 42U.
  - Rack 1: servidores (web, ldap, logs, bbdd, audio, video, jitsi).
  - Rack 2: switches, firewall, KVM, 2× SAI APC Smart-UPS RT 3000VA.
- **Alimentación:** doble PSU en servidores, dos circuitos independientes (A y B). Consumo estimado: 1.030 W. Autonomía SAI: 120 minutos.
- **Seguridad física:** lector biométrico + RFID, 4 cámaras IP PTZ con NVR, detección VESDA, extinción con gas Novec 1230, puerta blindada RC3, paredes de hormigón armado 20 cm, sensores de inundación.
- **PRL:** pasillos de 90 cm, iluminación de emergencia, señalización ISO 7010, EPI, límite de permanencia 30 min.
- **Nivel de disponibilidad:** Tier II (99,741 %).

---

### Logs centralizados (Módulo 0371)

- **Software:** rsyslog (preinstalado en Ubuntu).
- **Servidor central:** `ec2-logs` (10.0.1.201), puertos 514 TCP y UDP abiertos.
- **Configuración servidor:** módulos `imudp` e `imtcp` habilitados en `/etc/rsyslog.conf`. Plantilla `RemoteLogs` que organiza los logs por hostname y programa en `/var/log/remote/<HOSTNAME>/<PROGRAMA>.log`.
- **Configuración clientes:** línea `*.* @10.0.1.201:514` añadida en `/etc/rsyslog.conf` de cada máquina (automatizado con Ansible).
- **Logs recibidos:** logs de SSH, systemd, servicios propios (nginx, icecast, jellyfin, mariadb) de las 7 instancias.


---

### Servicio de streaming de audio (Módulo 0375)

- **Software:** Icecast2 + FFmpeg + Screen.
- **Formato:** MP3 a 128 kbps y OGG Vorbis a 128 kbps.
- **Configuración:** `/etc/icecast2/icecast.xml` con hostname (IP pública), contraseñas `pirineus`, puerto 8000, límite de 100 clientes.
- **Funcionamiento:** proceso `screen` ejecuta `ffmpeg` en bucle enviando `audio_prova.mp3` al mount point `/stream` (MP3) y `/stream.ogg` (OGG).
- **Acceso web:** `http://<IP-pública>:8000` — panel de estadísticas y reproductor embebido.
- **Validación:** stream activo con múltiples clientes simultáneos verificado en el panel de administración (`/admin/stats.xml`).

---

### Servicio de streaming de vídeo y videoconferencia (Módulo 0375)

**Streaming de vídeo:**
- Servidor NGINX con módulo RTMP. Ingesta por RTMP (puerto 1935), distribución HLS (puerto 8080).
- Configuración en `/etc/nginx/nginx.conf`: bloque `rtmp` con aplicación `live`, HLS activado con fragmentos de 3 segundos en `/tmp/hls`. Bloque `http` sirviendo `/tmp/hls` con cabeceras CORS.
- Reproductor web `player.html` con librería `hls.js`, accesible en `http://<IP>:8080/player.html`.
- Jellyfin instalado con script oficial, biblioteca de vídeos en `/opt/videos` (permisos `jellyfin:jellyfin`). Acceso web en puerto 8096.
- Vídeo de prueba: `willy.mp4` (codec H.264, formato MP4).

**Videoconferencia:**
- Jitsi Meet en contenedores Docker (`docker-compose`).
- Repositorio oficial `docker-jitsi-meet`, archivo `.env` con `PUBLIC_URL`, puertos 80/443, contraseñas generadas con `gen-passwords.sh`.
- Protocolo WebRTC, puerto UDP 10000 para medios.
- Acceso: `https://<IP-pública>`.

---

### Análisis de ancho de banda (Módulo 0375)

| Herramienta | Resultado |
|-------------|-----------|
| speedtest-cli | Download: 749,82 Mbit/s · Upload: 8,31 Mbit/s · Latencia: 10,617 ms |
| iperf3 (interno) | 313 Mbit/s estables, estabilización completa a partir del segundo 2 |

| Servicio | Download necesario / real | Upload necesario / real | Latencia límite / real | Estado |
|---------|--------------------------|------------------------|----------------------|--------|
| Audio MP3 | 0,2 / 749 Mbps | 0,2 / 8,31 Mbps | < 500 / 10,6 ms | Excelente |
| Vídeo 1080p HLS | 8 / 749 Mbps | 8 / 8,31 Mbps | < 200 / 10,6 ms | Muy justo |
| Videoconferencia | 4 / 749 Mbps | 4 / 8,31 Mbps | < 150 / 10,6 ms | Aceptable |

**Clasificación del sistema: ACCEPTABLE.** El cuello de botella es la subida (8,31 Mbps): soporta un único stream o videollamada simultánea, pero no múltiples emisores en directo.

**Propuestas de mejora:** instancia t3.medium (mayor ancho de banda), CDN para distribución de vídeo, políticas QoS para priorizar tráfico de videollamadas, monitorización en tiempo real.

---

### Base de datos (Módulo 0377)

- **SGBD:** MariaDB. Justificación: compatibilidad con MySQL, rendimiento, facilidad de uso, gratuito, soporte activo.
- **Hardening:** `mysql_secure_installation` — eliminación de usuarios anónimos y base de datos test, contraseña root robusta, acceso remoto restringido.
- **Modelo:** 9 entidades — `Departament`, `Empleat`, `Usuari_comunicacio`, `Grup_qualitat`, `Trucada`, `Video`, `Mesura_BW`, `Avis_auditoria`, `Control_backup`. Diagrama E/R disponible en [Google Drive](https://drive.google.com/file/d/1BxvITU8HYOXiMdQY0S00P1nkJsLVQZRA/view).
- **Código fuente SQL:** [Google Drive](https://drive.google.com/file/d/1UDlYD38ebCAn6BMpwJ99NEg2WSlSFCdf/view).

**Triggers implementados:**

| Trigger | Función |
|---------|---------|
| `tr_quota_minuts_mensuals` | Bloquea inserción de llamada si el usuario supera los minutos mensuales de su grupo. Registra en `Avis_auditoria`. |
| `tr_quota_trucades_diaries` | Bloquea inserción si el usuario supera el máximo de llamadas diarias. Registra en `Avis_auditoria`. |
| `tr_bloqueig_usuari` | Impide realizar o recibir llamadas si el usuario (originador o destinatario) está en estado `bloquejat`. |
| `tr_auditoria_empleat_update` | Registra en `Avis_auditoria` cualquier UPDATE sobre la tabla `Empleat`. |
| `tr_auditoria_empleat_delete` | Registra en `Avis_auditoria` cualquier DELETE sobre la tabla `Empleat`. |

**Roles y permisos:**

| Rol | Permisos principales | Restricciones |
|-----|---------------------|---------------|
| `admin` | Control total sobre todas las tablas. GRANT FILE. | Puede gestionar usuarios y ver avisos. |
| `vendes` | SELECT, INSERT, UPDATE en clientes, comandes, trucades. | Puede iniciar llamadas y consultar historial. |
| `administracio` | SELECT, INSERT, UPDATE en personal, departaments, grup_qualitat. | Sin acceso al sistema de videollamadas de clientes. |
| `treballador` | SELECT en vídeos, productes, configuració de trucades. | No puede modificar personal ni datos de clientes. |

**Script de creación de usuarios:**
- Bash — parámetros: usuario, contraseña, rol.
- Genera archivo `.sql` con `CREATE USER` y `GRANT` según el rol.
- Gestiona errores (usuario existente, rol no válido).
- Asigna privilegio `FILE` al root para backups.
- [Script en Google Drive](https://drive.google.com/file/d/1QA72P29FfZ3MkVSoOIPpU-VkvBzdVuJo/view).

**Backup automático:**
- Evento diario `ev_backup_diari` — exporta 7 tablas críticas a CSV en `/tmp/` con `SELECT INTO OUTFILE`.
- Tablas incluidas: `Empleat`, `Departament`, `Trucada`, `Usuari_comunicacio`, `Video`, `Mesura_BW`, `Grup_qualitat`.
- Registra cada ejecución en `Control_backup` con fecha, tablas y resultado.

---

### Seguridad y digitalización (Módulo 1665)

**Seguridad física:**
Sala blindada con control de acceso biométrico + RFID, videovigilancia con 4 cámaras PTZ y NVR local, detección de incendios VESDA, extinción con gas Novec 1230 (sin residuos, seguro para equipos), SAI con 2 horas de autonomía, puertas de emergencia con barra antipánico.

**Seguridad lógica:**
Autenticación SSH exclusivamente por clave pública/privada, usuario administrador `adminitb` (nunca root), Security Groups de AWS con política de mínimo privilegio, logs centralizados con rsyslog para detección de intrusiones, actualizaciones de seguridad automáticas, RAID 1 para el sistema operativo y RAID 5 para datos, copias de seguridad diarias cifradas.

**Importancia de los datos y cumplimiento normativo:**
La base de datos contiene datos personales de empleados (DNI, dirección, teléfono) sujetos al RGPD. Los triggers de auditoría garantizan la trazabilidad de accesos no autorizados. Las copias de seguridad automáticas aseguran la recuperación ante desastres. La arquitectura de roles diferenciados limita el principio de mínimo privilegio por perfil.

**Transformación digital:**
La solución moderniza Innovate Tech pasando de comunicaciones presenciales a un sistema integrado de streaming, videoconferencia y gestión de datos en la nube. Permite escalar los servicios según demanda, reducir costes operativos y garantizar la continuidad del negocio con una infraestructura automatizada y documentada.

---

## Vídeo demostración (3 minutos)

[Enlace al vídeo de YouTube]

El vídeo muestra:
1. **Introducción** 
2. **Demostración práctica:**
3. **Conclusión** 

---

## Tecnologías utilizadas

| Categoría | Tecnología |
|-----------|-----------|
| Nube | AWS EC2, VPC, Security Groups |
| Automatización | Ansible |
| Logs | rsyslog |
| Audio | Icecast2, FFmpeg, Screen |
| Vídeo | NGINX + módulo RTMP, HLS, hls.js, Jellyfin, FFmpeg |
| Videoconferencia | Jitsi Meet, Docker, Docker Compose, WebRTC |
| Base de datos | MariaDB |
| Directorios | OpenLDAP (slapd) |
| Transferencia de ficheros | OpenSSH SFTP con chroot |
| Servidor web | NGINX / Apache |
| Documentación | Markdown, GitHub |

---



## Autores 

| Miembro    | 
|------------|
| **Ayoub**  | 
| **Didac**  | 
| **Faiver** |

---

## Licencia

Este proyecto se entrega como trabajo académico para el ciclo formativo de **Administración de Sistemas Informáticos en Red (ASIX)** en el **Institut Tecnològic de Barcelona (ITB)**.  
Queda prohibida su reproducción total o parcial sin autorización expresa de los autores.



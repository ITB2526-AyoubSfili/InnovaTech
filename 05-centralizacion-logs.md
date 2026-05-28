# 5. Centralización de Logs — rsyslog

Esta sección cubre la implementación de un servidor centralizado de logs mediante **rsyslog** para recopilar los registros de todos los equipos del proyecto InnovateTech. Permite monitorizar incidentes, auditar accesos y centralizar la información de seguridad.

Servicio requerido por el módulo **0371** (CPD Cloud AWS) — Logs centralizados con agentes en todos los equipos.

---

## Arquitectura

| Servidor | IP Privada | Función |
|---|---|---|
| Logs central | 10.0.1.201 | Recibe y almacena logs de todos los equipos |
| Clientes | 10.0.1.60, 10.0.1.23, 10.0.1.208, 10.0.1.17, 10.0.1.209, 10.0.1.237 | Envían sus logs locales al servidor |

**Protocolo utilizado:** syslog sobre TCP/UDP puerto `514`.

---

## Configuración del Servidor de Logs (Máquina 10.0.1.201)

### Paso 1: Instalación

```bash
ubuntu@ip-10-0-1-201:~$ sudo apt update && sudo apt install rsyslog -y
```

---

### Paso 2: Habilitar Recepción de Logs Remotos

Editar el archivo `/etc/rsyslog.conf` y descomentar / añadir:

```bash
ubuntu@ip-10-0-1-201:~$ sudo nano /etc/rsyslog.conf
```

Localizar las siguientes líneas y descomentarlas o añadirlas:

```nginx
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```

Estas directivas habilitan la escucha en el puerto `514` tanto en TCP como UDP.

---

### Paso 3: Plantilla para Almacenar Logs por Equipo

Al final del mismo archivo, añadir:

```nginx
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
```

Esta configuración organiza los logs en directorios separados por nombre de host y programa, facilitando la búsqueda y auditoría.

---

### Paso 4: Crear Directorio y Ajustar Permisos

```bash
ubuntu@ip-10-0-1-201:~$ sudo mkdir -p /var/log/remote
ubuntu@ip-10-0-1-201:~$ sudo chown -R syslog:syslog /var/log/remote/
ubuntu@ip-10-0-1-201:~$ sudo chmod -R 755 /var/log/remote/
```

---

### Paso 5: Reiniciar el Servicio

```bash
ubuntu@ip-10-0-1-201:~$ sudo systemctl restart rsyslog
ubuntu@ip-10-0-1-201:~$ sudo systemctl status rsyslog
```

Salida esperada:

```
rsyslog.service - System Logging Daemon
   Loaded: loaded (/lib/systemd/system/rsyslog.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2026-05-18 10:22:15 UTC; 2s ago
```

> [!NOTE]
> Incluir captura de pantalla: `rsyslog-server-status.png` (mostrando `active (running)`)

---

### Paso 6: Verificar que Escucha en los Puertos 514

```bash
ubuntu@ip-10-0-1-201:~$ sudo ss -tlnp | grep 514
ubuntu@ip-10-0-1-201:~$ sudo ss -ulnp | grep 514
```

Salida esperada:

```
tcp   LISTEN  0  25  0.0.0.0:514  0.0.0.0:*  users:(("rsyslogd",pid=1234,fd=3))
udp   UNCONN  0  0   0.0.0.0:514  0.0.0.0:*  users:(("rsyslogd",pid=1234,fd=4))
```

---

## Configuración de los Clientes (Todos los Otros EC2)

A cada máquina cliente (audio, vídeo, BD, Jitsi, LDAP, web) ejecutar:

```bash
ubuntu@ip-10-0-1-60:~$ echo "*.* @10.0.1.201:514" | sudo tee -a /etc/rsyslog.conf
ubuntu@ip-10-0-1-60:~$ sudo systemctl restart rsyslog
```

> [!NOTE]
> rsyslog viene preinstalado en Ubuntu. Solo es necesario añadir la línea de envío al archivo de configuración.

Repetir este procedimiento en todas las máquinas clientes (10.0.1.23, 10.0.1.208, 10.0.1.17, 10.0.1.209, 10.0.1.237).

---

## Verificación

### Enviar un Mensaje de Prueba desde un Cliente

```bash
ubuntu@ip-10-0-1-60:~$ logger -n 10.0.1.201 -P 514 "Test log desde la máquina audio"
```

### Comprobar que ha Llegado al Servidor

```bash
ubuntu@ip-10-0-1-201:~$ sudo ls /var/log/remote/
```

La salida debe mostrar carpetas con los nombres/IPs de los clientes:

```
ip-10-0-1-201   ip-10-0-1-208   ip-10-0-1-23   ip-10-0-1-60
ip-10-0-1-209   ip-10-0-1-237
```

> [!NOTE]
> Incluir captura de pantalla: `logs-remote-dirs.png` (listado de carpetas por cada equipo cliente)

---

### Ver el Contenido de un Log Concreto

```bash
ubuntu@ip-10-0-1-201:~$ sudo cat /var/log/remote/ip-10-0-1-208/sshd.log
```

Salida esperada (ejemplo):

```
May 18 10:35:22 ip-10-0-1-208 sshd[1523]: Invalid user admin from 192.168.1.100 port 43210
May 18 10:35:25 ip-10-0-1-208 sshd[1524]: Failed password for invalid user admin from 192.168.1.100 port 43210 ssh2
May 18 10:36:01 ip-10-0-1-208 sshd[1525]: Accepted publickey for ubuntu from 10.0.1.201 port 54321 ssh2
```

> [!NOTE]
> Incluir captura de pantalla: `logs-contenido.png` (mostrando, por ejemplo, intentos de conexión SSH registrados)

---

### Estructura de Directorios Generada

```
/var/log/remote/
├── ip-10-0-1-60/
│   ├── icecast.log
│   ├── ffmpeg.log
│   └── sshd.log
├── ip-10-0-1-23/
│   ├── nginx.log
│   ├── jellyfin.log
│   └── sshd.log
├── ip-10-0-1-208/
│   ├── mariadb.log
│   ├── mysqld.log
│   └── sshd.log
└── ... (resto de máquinas)
```

---

## Problemas Detectados y Soluciones

| Problema | Solución |
|---|---|
| `Permission denied` en `/var/log/remote/` | `sudo chown -R syslog:syslog /var/log/remote/` |
| Los logs no llegan al servidor | Comprobar Security Group de AWS: abrir puerto `514` TCP/UDP |
| Espacio en disco al 100% (`/dev/root`) | Borrar logs antiguos, ampliar volumen AWS a 20 GB y ejecutar `growpart` + `resize2fs` |
| El cliente no envía nada | Verificar que rsyslog esté activo: `systemctl status rsyslog` |
| Conflicto de puertos (514 en uso) | Cambiar puerto en `/etc/rsyslog.conf` a `5514` y actualizar en todos los clientes |

---

## Comandos Útiles para Mantenimiento

| Comando | Descripción |
|---|---|
| `sudo systemctl restart rsyslog` | Reiniciar el servicio rsyslog |
| `sudo systemctl status rsyslog` | Ver estado del servicio |
| `sudo journalctl -u rsyslog -f` | Ver logs en tiempo real de rsyslog |
| `sudo find /var/log/remote -name "*.log" -mtime +7 -delete` | Borrar logs más antiguos de 7 días |
| `sudo du -sh /var/log/remote/` | Ver espacio total ocupado por los logs remotos |
| `sudo tail -f /var/log/remote/ip-10-0-1-60/sshd.log` | Monitorizar logs en tiempo real de un equipo específico |

---

## Auditoría y Monitorización

Con los logs centralizados, es posible realizar auditorías de seguridad más profundas:

**Detectar intentos de acceso fallidos:**

```bash
ubuntu@ip-10-0-1-201:~$ sudo grep "Failed password" /var/log/remote/*/sshd.log | wc -l
```

**Identificar qué usuario realizó cambios en un archivo:**

```bash
ubuntu@ip-10-0-1-201:~$ sudo grep "sudo" /var/log/remote/ip-10-0-1-208/sudo.log
```

**Búsqueda de errores críticos en todos los servicios:**

```bash
ubuntu@ip-10-0-1-201:~$ sudo grep -r "ERROR\|CRITICAL" /var/log/remote/
```

---

## Conclusión

La centralización de logs con rsyslog proporciona:

- **Visibilidad total** de eventos en toda la infraestructura
- **Trazabilidad completa** para auditoría y cumplimiento normativo
- **Detección rápida** de incidentes de seguridad
- **Facilidad de búsqueda** mediante la organización por máquina y servicio
- **Reducción de espacio** en discos individuales al centralizarse en un único servidor

Este sistema es fundamental para la **monitorización proactiva** y la **respuesta ante incidentes** en el entorno de producción.

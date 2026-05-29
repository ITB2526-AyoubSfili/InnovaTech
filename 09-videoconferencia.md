# 3. Servidor de Videoconferencia — Jitsi Meet + Docker

Esta sección cubre el despliegue de un servidor de videoconferencia autogestionado utilizando **Jitsi Meet** sobre **Docker Compose**. El sistema permite realizar videollamadas cifradas de extremo a extremo sin depender de servicios externos como Zoom o Google Meet, ejecutándose íntegramente en la instancia AWS EC2.

---

## Paso 1: Instalación de Docker y Docker Compose

Actualizamos el sistema e instalamos Docker junto con el plugin de Compose:

```bash
ubuntu@ip-10-0-1-60:~$ sudo apt update && sudo apt upgrade -y
ubuntu@ip-10-0-1-60:~$ sudo apt install docker.io docker-compose -y
```

Habilitamos el servicio Docker para que arranque automáticamente con el sistema:

```bash
ubuntu@ip-10-0-1-60:~$ sudo systemctl enable docker
ubuntu@ip-10-0-1-60:~$ sudo systemctl start docker
```

---

## Paso 2: Clonado del Repositorio Oficial de Jitsi Meet

Clonamos la plantilla oficial de despliegue Docker mantenida por el equipo de Jitsi:

```bash
ubuntu@ip-10-0-1-60:~$ git clone https://github.com/jitsi/docker-jitsi-meet.git
ubuntu@ip-10-0-1-60:~$ cd docker-jitsi-meet
```

---

## Paso 3: Configuración del Archivo de Entorno (`.env`)

El repositorio incluye una plantilla de configuración base que debemos copiar y editar:

```bash
ubuntu@ip-10-0-1-60:~/docker-jitsi-meet$ cp env.example .env
ubuntu@ip-10-0-1-60:~/docker-jitsi-meet$ nano .env
```

Las variables clave que se configuraron dentro del archivo `.env`:

```env
# IP pública de la instancia AWS EC2
PUBLIC_URL=https://32.194.168.28

# Contraseñas internas de comunicación entre contenedores
JICOFO_AUTH_PASSWORD=<contraseña_segura>
JVB_AUTH_PASSWORD=<contraseña_segura>
JIGASI_XMPP_PASSWORD=<contraseña_segura>
JIBRI_RECORDER_PASSWORD=<contraseña_segura>
JIBRI_XMPP_PASSWORD=<contraseña_segura>
```

> [!WARNING]
> Las contraseñas deben generarse con valores aleatorios y seguros. El propio repositorio oficial incluye un script para generarlas automáticamente:
> ```bash
> ubuntu@ip-10-0-1-60:~/docker-jitsi-meet$ ./gen-passwords.sh
> ```
> Ejecutar este script rellena automáticamente todos los campos de contraseña en el `.env`.

---

## Paso 4: Creación de los Directorios de Configuración

Jitsi Meet requiere una serie de directorios locales donde los contenedores almacenan su configuración persistente:

```bash
ubuntu@ip-10-0-1-60:~/docker-jitsi-meet$ mkdir -p ~/.jitsi-meet-cfg/{web,transcripts,prosody/config,prosody/prosody-plugins-custom,jicofo,jvb,jigasi,jibri}
```

---

## Paso 5: Apertura de Puertos en el Security Group de AWS

Para que el servicio sea accesible desde el exterior, se configuraron las siguientes reglas de entrada en el Security Group de la instancia EC2:

| Puerto | Protocolo | Uso |
|---|---|---|
| `80` | TCP | Redirección HTTP a HTTPS |
| `443` | TCP | Interfaz web segura (HTTPS) |
| `8443` | TCP | Señalización RTCP / STUN |
| `10000` | UDP | Tráfico de medios en tiempo real (audio/vídeo) |

> [!NOTE]
> El puerto `10000/UDP` es crítico para la calidad de la llamada. Si está cerrado, Jitsi Meet puede parecer funcionar pero el audio y el vídeo no llegarán correctamente a los participantes.

---

## Paso 6: Arranque de los Contenedores

Lanzamos todos los servicios de Jitsi Meet en segundo plano con Docker Compose:

```bash
ubuntu@ip-10-0-1-60:~/docker-jitsi-meet$ sudo docker-compose up -d
```

Verificamos que todos los contenedores están en ejecución:

```bash
ubuntu@ip-10-0-1-60:~/docker-jitsi-meet$ sudo docker-compose ps
```

Salida esperada:

```
Name                     Command               State           Ports
------------------------------------------------------------------------
jitsi-meet_jicofo_1     /init                  Up
jitsi-meet_jvb_1        /init                  Up      0.0.0.0:10000->10000/udp
jitsi-meet_prosody_1    /init                  Up
jitsi-meet_web_1        /init                  Up      0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
```

---

## Paso 7: Acceso y Comprobación del Servicio

Una vez los contenedores están activos, el servicio es accesible desde cualquier navegador:

| Recurso | URL |
|---|---|
| Interfaz Web Jitsi Meet | `https://32.194.168.28` |

Para crear una sala de videoconferencia basta con acceder a la URL e introducir un nombre de sala. El sistema no requiere registro ni instalación de software en el cliente.

> [!WARNING]
> Al acceder con IP pública sin certificado SSL firmado por una CA reconocida, el navegador mostrará un aviso de seguridad. En un entorno de producción real se recomienda configurar un dominio propio con un certificado **Let's Encrypt** mediante la variable `LETSENCRYPT_DOMAIN` en el `.env`.

---

## Comandos Utiles de Gestion

| Comando | Descripcion |
|---|---|
| `sudo docker-compose up -d` | Arrancar todos los contenedores en segundo plano |
| `sudo docker-compose down` | Detener y eliminar los contenedores |
| `sudo docker-compose restart` | Reiniciar el servicio completo |
| `sudo docker-compose logs -f` | Ver logs en tiempo real de todos los servicios |
| `sudo docker-compose ps` | Listar el estado de los contenedores |

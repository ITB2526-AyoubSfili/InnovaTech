# Servidor de Streaming de Vídeo — NGINX + RTMP + HLS

Esta sección cubre el despliegue de una arquitectura de distribución de vídeo de alta definición basada en el servidor **NGINX**. El sistema actúa como un nodo de ingesta mediante el protocolo **RTMP** (puerto `1935`) y realiza una transcodificación/segmentación en caliente al protocolo **HLS** (HTTP Live Streaming), almacenando los fragmentos `.ts` y la lista de reproducción `.m3u8` dinámicamente para ser servidos por HTTP a través del puerto `8080`.

---

### ¿Por qué HLS sobre RTMP para clientes web?

> [!NOTE]
> Aunque RTMP ofrece una latencia ultra baja, los navegadores web modernos **no soportan** la reproducción directa de este protocolo de forma nativa debido a la obsolescencia de Adobe Flash. Al convertir el flujo RTMP entrante a HLS, dividimos el vídeo en pequeños archivos segmentados transportables mediante HTTP estándar. Esto permite la compatibilidad absoluta con cualquier dispositivo móvil y navegador web utilizando librerías JavaScript como **hls.js**.

---

## Paso 1: Instalación de NGINX y el Módulo RTMP

Actualizamos los índices de paquetes e instalamos el servidor web NGINX junto con su módulo oficial de extensión para streaming en tiempo real (`libnginx-mod-rtmp`).

```bash
ubuntu@ip-10-0-1-60:~$ sudo apt update
ubuntu@ip-10-0-1-60:~$ sudo apt install nginx libnginx-mod-rtmp -y
```

---

## Paso 2: Creación del Directorio de Trabajo Temporal para HLS

Para optimizar la velocidad de lectura/escritura de los fragmentos de vídeo generados en tiempo real, se asigna la ruta `/tmp/hls`.

```bash
ubuntu@ip-10-0-1-60:~$ sudo mkdir -p /tmp/hls
ubuntu@ip-10-0-1-60:~$ sudo chown -R www-data:www-data /tmp/hls
```

---

## Paso 3: Configuración Avanzada de NGINX (`nginx.conf`)

Modificamos el archivo maestro de configuración de NGINX para definir la estructura de rendimiento, la inyección de módulos multimedia, el bloque de escucha RTMP y el servidor HTTP para la distribución de fragmentos web.

```bash
ubuntu@ip-10-0-1-60:~$ sudo nano /etc/nginx/nginx.conf
```

Configuración completa aplicada al entorno:

```nginx
# Estructura Base: Configuración de Rendimiento y Seguridad
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024; # Permite manejar miles de conexiones simultáneas
}

# Bloque RTMP: Ingesta de la señal en directo y conversión a HLS
rtmp {
    server {
        listen 1935;     # Puerto de escucha estándar para RTMP
        chunk_size 4000;

        application live {
            live on;     # Habilita la transmisión en vivo

            # Activación de la fragmentación HTTP Live Streaming (HLS)
            hls on;
            hls_path /tmp/hls;         # Directorio donde se guardan los fragmentos
            hls_fragment 3s;           # Duración de cada fragmento de vídeo (.ts)
            hls_playlist_length 60s;   # Ventana de tiempo total de la lista de reproducción
        }
    }
}

# Bloque HTTP: Entrega web sin retrasos ni bloqueos de seguridad (CORS)
http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    server {
        listen 8080;     # Puerto configurado para la distribución multimedia
        server_name localhost;

        # Ubicación de los fragmentos multimedia y configuración de políticas CORS
        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /tmp;

            # Cabeceras de control de acceso indispensables para evitar bloqueos del navegador
            add_header Access-Control-Allow-Origin *;
            add_header Cache-Control no-cache;
        }

        # Servir el reproductor web HTML5 integrado
        location / {
            root /var/www/html;
            index player.html;
        }
    }
}
```

---

## Paso 4: Verificación Sintáctica y Reinicio de NGINX

Antes de reiniciar el proceso de producción, validamos que no existan errores de sintaxis en la configuración:

```bash
ubuntu@ip-10-0-1-60:~$ sudo nginx -t
```

Salida esperada:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Procedemos al reinicio y control del estado operativo del daemon:

```bash
ubuntu@ip-10-0-1-60:~$ sudo systemctl restart nginx
ubuntu@ip-10-0-1-60:~$ sudo systemctl status nginx
```

---

## Paso 5: Descarga y Verificación del Vídeo Base en MP4

Descargamos un archivo MP4 estándar para simular la entrada de vídeo en tiempo real hacia los sockets de ingesta de NGINX:

```bash
ubuntu@ip-10-0-1-60:~$ wget -O /tmp/video_prova.mp4 \
  "https://sample-videos.com/video321/mp4/720/big_buck_bunny_720p_1mb.mp4"
```

Verificamos las propiedades del contenedor multimedia descargado:

```bash
ubuntu@ip-10-0-1-60:~$ ffmpeg -i /tmp/video_prova.mp4
```

---

## Paso 6: Transmisión RTMP Continua en Segundo Plano (Uso de `screen`)

Para emular una cámara o encoder (como OBS Studio) enviando flujo continuo al servidor las 24 horas del día sin depender de la sesión SSH, recurrimos a **`screen`**.

**1. Crear e ingresar a una sesión virtual:**

```bash
ubuntu@ip-10-0-1-60:~$ screen -S video
```

**2. Ejecutar la inyección multimedia por RTMP:**

El parámetro `-re` lee el vídeo a velocidad nativa, `-stream_loop -1` define un bucle infinito, `-vcodec libx264` codifica en H.264 con perfil rápido (`-preset veryfast`), `-acodec aac` transforma el audio a AAC, y se formatea la salida como FLV hacia el punto de montaje RTMP:

```bash
ffmpeg -re -stream_loop -1 -i /tmp/video_prova.mp4 \
  -vcodec libx264 -preset veryfast -maxrate 3000k -bufsize 6000k \
  -acodec aac -b:a 128k \
  -f flv rtmp://localhost:1935/live/stream
```

**3. Desasociar la sesión:** `Ctrl + A` → `D`

El stream de vídeo ya está siendo procesado y fragmentado continuamente por NGINX.

---

# Servidor de Vídeo en Streaming — Jellyfin

## Descripció general

**Jellyfin** és un servidor de mediateca gratuït i de codi obert que permet gestionar, organitzar i visualitzar pel·lícules, sèries, música i fotos en qualsevol dispositiu amb navegador web o aplicació mòbil.

En aquest projecte, Jellyfin actua com a **catàleg de vídeos a demanda (VOD)** complementari al sistema de streaming en directe (RTMP/HLS). 

---


### 2. Descarregar i executar el script oficial

Jellyfin ofereix un script d'instal·lació oficial que simplifica el procés:

```bash
# Descarregar l'script
curl https://repo.jellyfin.org/install-debuntu.sh -o install-jellyfin.sh

# Fer-lo executable
chmod +x install-jellyfin.sh

# Executar l'instal·lador 
sudo ./install-jellyfin.sh
```

El script:
- Afegeix el repositori oficial de Jellyfin
- Instal·la les dependències necessàries
- Instal·la el paquet `jellyfin`
- Crea l'usuari del sistema `jellyfin`
- Configura el servei per iniciar automàticament

### 3. Habilitar i arrancar el servei

```bash
# Assegurar que el servei està habilitat per iniciar-se al boot
sudo systemctl enable jellyfin

# Iniciar el servei
sudo systemctl start jellyfin



## Configuració

### 1. Crear el directori de la biblioteca

Jellyfin necessita un directori on emmagatzemar els vídeos. Per seguretat i organització, usarem `/opt/videos`:

```bash
# Crear el directori
sudo mkdir -p /opt/videos

# Canviar propietari a l'usuari jellyfin (qui executa el servei)
sudo chown -R jellyfin:jellyfin /opt/videos

# Establir permisos (755: propietari pot llegir/escriure/executar, altres només llegir)
sudo chmod -R 755 /opt/videos
```

### 2. Afegir vídeos al directori

Per aquest projecte, usem el vídeo de prova `willy.mp4`:

```bash
# Copiar el vídeo al directori (des d'on tinguis el fitxer)
sudo cp ~/willy.mp4 /opt/videos/


# Assegurar que els permisos són correctes
sudo chown jellyfin:jellyfin /opt/videos/willy.mp4
sudo chmod 644 /opt/videos/willy.mp4
```


### 3. Accedir a la interfície web

Obrir el navegador i anar a:

```
http://<IP-pública-ec2-video>:8096
```


---

## Configuració de la biblioteca dins de Jellyfin

### 1. Configuració inicial (wizard)

En la primera connexió, Jellyfin mostra un wizard de configuració:

1. **Idioma:** Seleccionar Català o Anglès
2. **Metadades:** Activar la descàrrega automàtica de cartells, sinopsis i informació
3. **Usuari admin:** Crear l'usuari administrador (opcional contrasenya)

### 2. Afegir la carpeta de biblioteca

Una vegada configurat:

1. Anar a **Settings** → **Libraries**
2. Clicar **+ Add Library**
3. Seleccionar tipo: **Movies** (pel·lícules)
4. Clicar **Add Folder**
5. Navegar a `/opt/videos` i seleccionar-la
6. Clicar **OK**
7. Clicar **Add** per crear la biblioteca

### 3. Escaneig de la biblioteca

Jellyfin automàticament:
- Escanejarà els fitxers MP4 al directori
- Baixarà metadades (títol, any, gènere, director) des de bases de dades externes
- Descarregarà imatges (cartell, fons)
- Crearà miniatures per a vista ràpida

El procés pot trigar segons la quantitat de vídeos. Veure el progrés a **Settings** → **Scheduled Tasks** → **Scan Library**.

---

## Accés i reproducció

### 1. Interfície web principal

```
http://<IP-pública>:8096/
```

Mostra:
- **Home:** últims vídeos afegits, recomanacions
- **Libraries:** accés a pel·lícules, sèries, música
- **Explore:** cerca avançada per títol, gènere, any

### 2. Cercar i reproduir un vídeo

1. Clicar a **Libraries** → **Movies**
2. Trobar "Willy" a la llista
3. Clicar en el cartell per obrir la fitxa del vídeo
4. Clicar **Play** per iniciar la reproducció

---

## Permisos i seguretat


### Accés a la web

Jellyfin **no requereix autenticació per defecte** en la biblioteca pública. Per afegir seguretat:

1. Anar a **Settings** → **Users** → **Create User**
2. Crear usuari `client` amb contrasenya
3. Assignar permisos específics per biblioteca

---

## Manteniment

### Comprovar l'estat del servei

```bash
# Ver si el servei està actiu
sudo systemctl status jellyfin

# Veure els últims logs
sudo journalctl -u jellyfin -n 50

# Veure logs en temps real
sudo journalctl -u jellyfin -f
```

### Reiniciar Jellyfin

```bash
sudo systemctl restart jellyfin
```

### Afegir més vídeos

```bash
# Copiar vídeos al directori
sudo cp ~/pelicula.mp4 /opt/videos/

# Canviar permisos
sudo chown jellyfin:jellyfin /opt/videos/pelicula.mp4
sudo chmod 644 /opt/videos/pelicula.mp4

# Desde la web: Settings → Libraries → Scan Library (o esperar que ho faci automàticament)
```

---




## Resolució de problemes

| Problema | Causa | Solució |
|----------|-------|---------|
| **Port 8096 no accessible** | Firewall bloquejat | Obrir port 8096 en Security Group AWS |
| **"Library empty"** | Cap vídeo a `/opt/videos` | Copiar fitxers i escannejar biblioteca |
| **Permisos denied** | Permisos incorrectes | `sudo chown -R jellyfin:jellyfin /opt/videos` |
| **Reproducció lenta** | Bitrate alt o CPU insuficient | Reduir resolució o activar transcoding |
| **No s'encarrega metadades** | Jellyfin sense accés a internet | Verificar connectivitat; descarregar XML de metadades manualment |

---


---

## Conclusió

Jellyfin proporciona una solució gratuïta i completa per a la gestió de videoteca en streaming. Integrat amb NGINX RTMP + HLS per a transmissions en directe, ofereix a **Innovate Tech** una plataforma multifacètica per a distribució de continguts audiovisuals.


>
> | Puerto | Protocolo | Uso |
> |---|---|---|
> | `1935` | TCP | Ingesta RTMP |
> | `8080` | TCP | Servicio Web de Vídeo HLS |

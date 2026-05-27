# 🎬 Servidor de Streaming de Vídeo — NGINX + RTMP + HLS

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

## Paso 7: Despliegue del Reproductor Web HTML5 y `hls.js`

Para que cualquier usuario pueda consumir la transmisión en vivo, creamos una interfaz web moderna. El archivo se programa en `/tmp/player.html` y luego se mueve a la raíz del servidor `/var/www/html/`.

**1. Asegurar la ruta del servidor web:**

```bash
ubuntu@ip-10-0-1-60:~$ sudo mkdir -p /var/www/html
```

**2. Escribir el código de la interfaz web:**

```bash
ubuntu@ip-10-0-1-60:~$ nano /tmp/player.html
```

Código fuente de `player.html` incorporando el motor JavaScript de descompresión HLS:

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>InnovateTech - Reproductor de Vídeo HLS en Vivo</title>
    <script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
    <style>
        body { font-family: Arial, sans-serif; background: #1e1e24; color: #fff; text-align: center; padding: 40px; }
        video { background: #000; border: 3px solid #3a3a45; border-radius: 8px; box-shadow: 0 4px 10px rgba(0,0,0,0.5); width: 100%; max-width: 800px; }
        .status-container { margin-top: 15px; font-size: 14px; color: #a0a0b0; }
    </style>
</head>
<body>
    <h1>🎬 Servidor de Vídeo en Streaming en Vivo (AWS EC2)</h1>
    <p>Proyecto ASIX - Transmisión en tiempo real utilizando NGINX + RTMP + HLS</p>

    <video id="video" controls autoplay muted></video>

    <div class="status-container">
        <p>Dirección del Punto de Montaje: <code style="color: #4af;">http://32.194.168.28:8080/hls/stream.m3u8</code></p>
    </div>

    <script>
        var video = document.getElementById('video');
        // Ruta absoluta apuntando a la IP pública de la instancia AWS EC2 en el puerto 8080
        var videoSrc = 'http://32.194.168.28:8080/hls/stream.m3u8';

        if (Hls.isSupported()) {
            var hls = new Hls();
            hls.loadSource(videoSrc);
            hls.attachMedia(video);
            hls.on(Hls.Events.MANIFEST_PARSED, function() {
                video.play();
            });
        }
        else if (video.canPlayType('application/vnd.apple.mpegurl')) {
            // Soporte nativo para navegadores Safari / iOS
            video.src = videoSrc;
            video.addEventListener('loadedmetadata', function() {
                video.play();
            });
        }
    </script>
</body>
</html>
```

**3. Mover el reproductor a la ubicación definitiva servida por NGINX:**

```bash
ubuntu@ip-10-0-1-60:~$ sudo mv /tmp/player.html /var/www/html/player.html
ubuntu@ip-10-0-1-60:~$ sudo chown www-data:www-data /var/www/html/player.html
```

---

## Paso 8: Comprobación Técnica Final del Servicio

Cualquier cliente remoto puede conectarse a la interfaz multimedia cargando la IP pública de la instancia en su navegador web:

| Recurso | URL |
|---|---|
| 🌐 Reproductor Web | `http://32.194.168.28:8080/player.html` |
| 📡 Punto de Montaje HLS | `http://32.194.168.28:8080/hls/stream.m3u8` |
| 📥 Ingesta RTMP | `rtmp://32.194.168.28:1935/live/stream` |

> [!WARNING]
> Si la página del reproductor carga la estructura pero el cuadro de vídeo se queda en negro o emite un error de red en la consola del desarrollador, verifique que los **Security Groups** de su instancia AWS EC2 tengan explícitamente abiertos los puertos de entrada:
>
> | Puerto | Protocolo | Uso |
> |---|---|---|
> | `1935` | TCP | Ingesta RTMP |
> | `8080` | TCP | Servicio Web de Vídeo HLS |

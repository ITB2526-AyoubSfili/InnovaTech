# Servidor de Streaming de Audio — Icecast2 + FFmpeg

Esta sección cubre el despliegue de un servidor de radiodifusión o streaming de audio utilizando **Icecast2** como servidor de streaming/distribución y **FFmpeg** como codificador de medios en tiempo real (fuente multimedia).

---

## Paso 1: Actualización del Sistema e Instalación de Paquetes

Es fundamental asegurar que los índices de repositorios locales estén actualizados antes de instalar herramientas multimedia avanzadas para prevenir conflictos de dependencias rotas o incompatibilidades de bibliotecas.

```bash
ubuntu@ip-10-0-1-60:~$ sudo apt update && sudo apt upgrade -y
ubuntu@ip-10-0-1-60:~$ sudo apt install icecast2 ffmpeg -y
```

---

## Paso 2: Configuración del Servidor Icecast2

El archivo de configuración principal controla las directivas del sistema, los límites de clientes, puertos de escucha y, de forma más crítica, las credenciales de transmisión y administración.

```bash
ubuntu@ip-10-0-1-60:~$ sudo nano /etc/icecast2/icecast.xml
```

> [!WARNING]
> Dentro de este archivo XML, es obligatorio modificar los bloques internos correspondientes a las contraseñas de origen (`<source-password>`), relevo (`<relay-password>`) y administración (`<admin-password>`). Para este proyecto de laboratorio, todas se definieron con el valor unificado **`pirineus`**. Esto permitirá la autenticación del codificador FFmpeg.

---

## Paso 3: Habilitación y Arranque del Servicio

Para garantizar que el demonio de Icecast2 se añada al gestor de arranque del sistema (systemd) y se inicialice inmediatamente:

```bash
ubuntu@ip-10-0-1-60:~$ sudo systemctl enable icecast2
ubuntu@ip-10-0-1-60:~$ sudo systemctl start icecast2
ubuntu@ip-10-0-1-60:~$ sudo systemctl status icecast2
```

Si el despliegue es exitoso, la salida en consola reflejará un estado activo y en ejecución:

```
icecast2.service - LSB: Icecast streaming media server
   Loaded: loaded (/etc/init.d/icecast2; generated)
   Active: active (running) since Mon 2026-05-18 06:18:21 UTC; 0min ago
```

> [!NOTE]
> Al iniciar por primera vez, el archivo de logs de Icecast2 puede emitir alertas de configuración (`WARN CONFIG/parse root warning`) indicando que no se han personalizado el parámetro de ubicación (`location`, por defecto `Earth`) ni el contacto del administrador (`admin contact`, por defecto `icemaster@localhost`). Estas alertas son puramente informativas y **no afectan la estabilidad de la transmisión**.

---

## Paso 4: Descarga del Contenido Multimedia de Prueba

Para realizar las pruebas de transmisión en vivo, descargamos un archivo de audio base al almacenamiento local de la instancia de AWS:

```bash
ubuntu@ip-10-0-1-60:~$ wget -O audio_prova2.mp3 "https://www.youtube.com/watch?v=5qpv0qR1eYY&list=RDSqpv0qRieYY&start_radio=1"
```

---

## Paso 5: Persistencia y Transmisión Continua en Bucle (Uso de `screen`)

FFmpeg normalmente se interrumpe si la sesión SSH se cierra o sufre un timeout. Para dotar al sistema de persistencia en segundo plano las 24 horas del día, se utiliza la herramienta **`screen`**, que despliega terminales virtuales independientes desasociadas del ciclo de vida de la sesión SSH.

###  Flujo de Streaming — Formato MP3

**1. Crear e ingresar a una sesión virtual dedicada:**

```bash
ubuntu@ip-10-0-1-60:~$ screen -S mp3
```

**2. Ejecutar la tubería multimedia de FFmpeg:**

El parámetro `-re` obliga a leer el archivo respetando la tasa de refresco en tiempo real, `-stream_loop -1` configura un bucle infinito, `-acodec libmp3lame` procesa la codificación con el códec MP3 Lame a 128 kbps y envía el flujo al puerto `8000` con el punto de montaje `/stream`:

```bash
ffmpeg -re -stream_loop -1 -i audio_prova.mp3 \
  -acodec libmp3lame -b:a 128k \
  -f mp3 icecast://source:pirineus@localhost:8000/stream
```

**3. Desasociar la pantalla:** `Ctrl + A` → `D`

La transmisión continuará operando de manera transparente en la instancia AWS.

---

###  Flujo de Streaming — Formato OGG (Vorbis)

Icecast también soporta nativamente el contenedor OGG con compresión Vorbis.

**1. Crear e ingresar a una sesión virtual independiente:**

```bash
ubuntu@ip-10-0-1-60:~$ screen -S ogg
```

**2. Ejecutar la codificación OGG:**

```bash
ffmpeg -re -stream_loop -1 -i audio_prova.mp3 \
  -acodec libvorbis -b:a 128k \
  -content_type application/ogg \
  icecast://source:pirineus@localhost:8000/stream.ogg
```
**2.1 Ejecutar la codificación AAC:**

```bash
screen -dmS aac-stream ffmpeg -re -stream_loop -1 -i ~/audio_prova.mp3 -vn -acodec aac -ab 128k -f adts icecast://source:pirineus@$IP_PUBLICA:8000/stream.aac
```

**3. Desasociar la pantalla:** `Ctrl + A` → `D`

> [!WARNING]
> Al codificar hacia un punto de montaje OGG, FFmpeg puede alertar con el mensaje `Streaming Ogg but appropriate content type NOT set!`. Es fundamental declarar explícitamente `-content_type application/ogg`. De lo contrario, los navegadores y reproductores de los clientes finales podrían fallar al procesar el flujo de audio entrante.

---

###  Comandos Útiles para el Control de Sesiones de `screen`

| Comando | Descripción |
|---|---|
| `screen -ls` | Listar todas las terminales virtuales activas |
| `screen -r mp3` | Reconectarse a la sesión del stream MP3 |
| `screen -r ogg` | Reconectarse a la sesión del stream OGG |

```bash
# Listar todas las terminales virtuales activas en el sistema
ubuntu@ip-10-0-1-60:~$ screen -ls

# Volver a conectarse a la sesión interactiva del stream MP3
ubuntu@ip-10-0-1-60:~$ screen -r mp3

# Volver a conectarse a la sesión interactiva del stream OGG
ubuntu@ip-10-0-1-60:~$ screen -r ogg
```

---

## Paso 6: Monitoreo y Panel de Control de Administración

Una vez que FFmpeg se encuentra inyectando datos de forma continua, el estado y las métricas de consumo de los oyentes pueden visualizarse a través del servidor web integrado en Icecast.

| Recurso | URL |
|---|---|
|  Panel Web de Estado | `http://ip:8000/` |
|  Estadísticas XML | `http://ip:8000/admin/stats.xml` |

### Métricas disponibles en `stats.xml`

A través del archivo de estadísticas de administración, un ingeniero DevOps puede extraer las siguientes métricas en tiempo real:

| Campo XML | Descripción | Ejemplo |
|---|---|---|
| `<client_connections>` | Conexiones totales administradas por el socket | `93` |
| `<listeners>` | Oyentes sintonizados concurrentemente | `1` |
| `<source_info>` | Parámetros de codificación de la fuente activa | `audio/mpeg, 128 kbps, Lavf/60.16.100` |

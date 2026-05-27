1. Servidor de Streaming de Audio (Icecast2 + FFmpeg)
Esta sección cubre el despliegue de un servidor de radiodifusión o streaming de audio utilizando Icecast2 como servidor de streaming/distribución y FFmpeg como codificador de medios en tiempo real (fuente multimedia).

Paso 1: Actualización del Sistema e Instalación de Paquetes
Es fundamental asegurar que los índices de repositorios locales estén actualizados antes de instalar herramientas multimedia avanzadas para prevenir cualquier conflicto de dependencias rotas o incompatibilidades de bibliotecas.

Bash
ubuntu@ip-10-0-1-60:~$ sudo apt update && sudo apt upgrade -y
ubuntu@ip-10-0-1-60:~$ sudo apt install icecast2 ffmpeg -y
Paso 2: Configuración del Servidor Icecast2
El archivo de configuración principal controla las directivas del sistema, los límites de clientes, puertos de escucha y, de forma más crítica, las credenciales de transmisión y administración.

Bash
ubuntu@ip-10-0-1-60:~$ sudo nano /etc/icecast2/icecast.xml
[!WARNING]
Dentro de este archivo XML, es obligatorio modificar los bloques internos correspondientes a las contraseñas de origen (<source-password>), relevo (<relay-password>) y administración (<admin-password>). Para este proyecto de laboratorio, todas se definieron con el valor unificado pirineus. Esto permitirá la autenticación del codificador FFmpeg.

Paso 3: Habilitación y Arranque del Servicio
Para garantizar que el demonio de Icecast2 se añada al gestor de arranque del sistema (systemd) y se inicialice inmediatamente de forma nativa o mediante compatibilidad SysV, ejecutamos:

Bash
ubuntu@ip-10-0-1-60:~$ sudo systemctl enable icecast2
ubuntu@ip-10-0-1-60:~$ sudo systemctl start icecast2
ubuntu@ip-10-0-1-60:~$ sudo systemctl status icecast2
Si el despliegue es exitoso, la salida en consola reflejará un estado activo y en ejecución:

Plaintext
Icecast2.service - LSB: Icecast streaming media server
   Loaded: loaded (/etc/init.d/icecast2; generated)
   Active: active (running) since Mon 2026-05-18 06:18:21 UTC; 0min ago
[!NOTE]
Al iniciar por primera vez, el archivo de logs de Icecast2 puede emitir alertas de configuración (WARN CONFIG/parse root warning) indicando que no se han personalizado el parámetro de ubicación (location, por defecto Earth) ni el contacto del administrador (admin contact, por defecto icemaster@localhost). Estas alertas son puramente informativas y no afectan la estabilidad de la transmisión.

Paso 4: Descarga del Contenido Multimedia de Prueba
Para realizar las pruebas de transmisión en vivo, procedemos a descargar un archivo de audio base al almacenamiento local de la instancia de AWS:

Bash
ubuntu@ip-10-0-1-60:~$ wget -O audio_prova2.mp3 "[https://www.youtube.com/watch?v=5qpv0qR1eYY&list=RDSqpv0qRieYY&start_radio=1](https://www.youtube.com/watch?v=5qpv0qR1eYY&list=RDSqpv0qRieYY&start_radio=1)"
Paso 5: Persistencia y Transmisión Continua en Bucle (Uso de screen)
El software de codificación FFmpeg normalmente se interrumpe si la sesión interactiva de SSH se cierra o sufre un timeout. Para solventar este problema y dotar al sistema de persistencia en segundo plano las 24 horas del día, se utiliza la herramienta screen. Esto despliega terminales virtuales independientes desasociadas del ciclo de vida de la sesión SSH del usuario.

Flujo de Streaming para Formato MP3
Crear e ingresar a una sesión virtual dedicada llamada mp3:

Bash
ubuntu@ip-10-0-1-60:~$ screen -S mp3
Ejecutar la tubería multimedia de FFmpeg. El parámetro -re obliga a leer el archivo respetando la tasa de refresco en tiempo real (nativo), -stream_loop -1 configura un bucle infinito para el archivo, -acodec libmp3lame procesa la codificación de audio con el códec MP3 Lame a una tasa constante de 128 kbps (-b:a 128k) enviando el flujo directo al puerto 8000 con el punto de montaje /stream:

Bash
ffmpeg -re -stream_loop -1 -i audio_prova.mp3 -acodec libmp3lame -b:a 128k -f mp3 icecast://source:pirineus@localhost:8000/stream
Desasociar la pantalla: Presionar la combinación de teclas Ctrl + A seguido de D. La transmisión continuará operando de manera transparente en la instancia AWS.

Flujo de Streaming para Formato OGG (Vorbis)
Icecast también soporta nativamente el contenedor multimedia OGG con compresión Vorbis. Para lanzar esta transmisión en paralelo:

Crear e ingresar a una sesión virtual independiente llamada ogg:

Bash
ubuntu@ip-10-0-1-60:~$ screen -S ogg
Ejecutar la codificación OGG: Cambiamos el códec a libvorbis, el formato de salida a ogg, la ruta de montaje a /stream.ogg e inyectamos el encabezado HTTP correspondiente:

Bash
ffmpeg -re -stream_loop -1 -i audio_prova.mp3 -acodec libvorbis -b:a 128k -content_type application/ogg icecast://source:pirineus@localhost:8000/stream.ogg
Desasociar la pantalla: Presionar Ctrl + A y luego D.

[!WARNING]
Al codificar hacia un punto de montaje OGG en Icecast, FFmpeg puede alertar con el mensaje Streaming Ogg but appropriate content type NOT set!. Es fundamental declarar explícitamente -content_type application/ogg en los parámetros de la dirección de red. De lo contrario, los navegadores y reproductores de los clientes finales podrían fallar al procesar e interpretar el flujo de audio entrante.

Comandos Útiles para el Control de Sesiones de Screen
Para monitorear o reasociar las tareas multimedia en segundo plano desde el servidor:

Bash
# Listar todas las terminales virtuales activas en el sistema
ubuntu@ip-10-0-1-60:~$ screen -ls

# Volver a conectarse a la sesión interactiva del stream MP3
ubuntu@ip-10-0-1-60:~$ screen -r mp3

# Volver a conectarse a la sesión interactiva del stream OGG
ubuntu@ip-10-0-1-60:~$ screen -r ogg
Paso 6: Monitoreo y Panel de Control de Administración
Una vez que FFmpeg se encuentra inyectando datos de forma continua, el estado y las métricas de consumo de los oyentes pueden ser visualizados externamente a través del servidor web integrado en Icecast.

URL del Panel Web de Estado: http://32.194.168.28:8000/

URL de Estadísticas XML Estructuradas: http://32.194.168.28:8000/admin/stats.xml

A través del archivo de estadísticas de administración (stats.xml), un ingeniero DevOps puede extraer métricas esenciales en tiempo real:

<client_connections>: Conexiones totales administradas por el socket (ej. 93).

<listeners>: Oyentes sintonizados actualmente de manera concurrente (ej. 1).

<source mount="/stream">: Muestra los parámetros de codificación de la fuente multimedia activa (audio/mpeg, 128 kbps, con un agente de origen Lavf/60.16.100).
"""

# 🎵 Servidor de Streaming de Audio (Icecast2)

El objetivo de estas pruebas es comprobar que el servidor Icecast2 funciona correctamente, que la transmisión de audio está activa y que los clientes pueden conectarse al stream.

---

## ✅ 1. Transmisión de audio con FFmpeg

Se inició la transmisión en bucle infinito hacia el servidor Icecast2 a 128 kbps.

### Comando utilizado

```bash
ffmpeg -re -stream_loop -1 -i audio_prova.mp3 \
  -vn -acodec libmp3lame -ab 128k -ar 44100 \
  -f mp3 icecast://source:pirineus@localhost:8000/stream
```

### Evidencia

<img width="771" height="576" alt="audio-ffmpeg" src="https://github.com/user-attachments/assets/c733032c-4af6-4633-88c3-56d5b19e52b4" />



## ✅ 2. Panel de administración de Icecast2

Se comprobó el acceso al panel web de administración desde un navegador externo.

### URL de acceso

```
http://32.194.168.28:8000/admin/
```

### Evidencia

<img width="928" alt="Panel Icecast2 Admin mostrando Global Server Stats: 2 clientes, Icecast 2.4.4, host 32.194.168.28" src="img/audio-icecast-panel.png" />

---

## ✅ 3. Verificación de clientes conectados al stream

Se comprobó que múltiples clientes pudieran conectarse al stream simultáneamente.

### URL

```
http://32.194.168.28:8000/admin/listclients.xsl?mount=/stream
```

### Evidencia

<img width="928" alt="Listener Stats mostrando 3 clientes conectados desde 79.117.174.171 con tiempos de 377s, 96s y 21s" src="img/audio-clientes.png" />

---

# 📊 Conclusión

Las pruebas realizadas permitieron verificar el correcto funcionamiento del servidor de streaming de audio.

Se confirmó:

* Transmisión de audio en bucle activa a 128 kbps con FFmpeg.
* Panel de administración Icecast2 accesible y con estadísticas correctas.
* Múltiples clientes capaces de conectarse y escuchar el stream simultáneamente.

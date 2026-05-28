# 04 - Servidor RTMP i Streaming HLS

## Introducció

Tot i que **RTMP** ofereix una latència molt baixa, els navegadors web moderns no suporten la reproducció directa d’aquest protocol de forma nativa a causa de l’obsolescència d’Adobe Flash.

Per solucionar-ho, el flux RTMP es converteix a **HLS (HTTP Live Streaming)**, dividint el vídeo en petits fragments transportables via HTTP estàndard. Això permet compatibilitat total amb navegadors i dispositius mòbils utilitzant biblioteques JavaScript com `hls.js`.

---

## Instal·lació de NGINX i el Mòdul RTMP

El servidor de streaming s’ha desplegat sobre una instància EC2 (`ip-10-0-1-60`) utilitzant **NGINX** amb el mòdul oficial RTMP.

### Pas 1: Instal·lació de NGINX i RTMP

```bash
sudo apt update
sudo apt install nginx libnginx-mod-rtmp -y
```

---

## Creació del Directori Temporal HLS

Per emmagatzemar els fragments `.ts` i les playlists `.m3u8` generades en temps real, es crea el directori temporal `/tmp/hls`.

### Pas 2: Creació del directori

```bash
sudo mkdir -p /tmp/hls
sudo chown -R www-data:www-data /tmp/hls
```

---

## Configuració Avançada de NGINX

Es modifica el fitxer principal de configuració de NGINX per habilitar:

- El servidor RTMP
- La generació automàtica d’HLS
- La distribució HTTP dels fragments multimèdia

### Pas 3: Edició del fitxer nginx.conf

```bash
sudo nano /etc/nginx/nginx.conf
```

Configuració aplicada:

```nginx
worker_processes auto;

events {
    worker_connections 1024;
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

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile on;
    tcp_nopush on;

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
    }
}
```

---

## Reinici i Verificació del Servei

Una vegada aplicada la configuració, es valida la sintaxi i es reinicia NGINX.

### Pas 4: Validació i reinici

```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx
```

---

## Emissió RTMP

Per publicar el flux de vídeo cap al servidor RTMP es pot utilitzar **OBS Studio**.

### Configuració d’OBS

| Paràmetre | Valor |
|----------|--------|
| Service | Custom |
| Server | rtmp://IP_PUBLICA/live |
| Stream Key | stream |

Exemple:

```text
rtmp://3.219.224.163/live
```

Clau d’stream:

```text
stream
```

---

## Reproducció HLS des del navegador

Una vegada iniciada l’emissió RTMP, NGINX genera automàticament:

- Fragments `.ts`
- Playlist `.m3u8`

URL de reproducció:

```text
http://3.219.224.163:8080/hls/stream.m3u8
```

---

## Verificació dels Fitxers HLS

Per comprovar que el servidor està generant correctament els fragments multimèdia:

### Pas 5: Comprovació dels fitxers generats

```bash
ls -lah /tmp/hls
```

Resultat esperat:

```text
stream.m3u8
stream0.ts
stream1.ts
stream2.ts
```

---

## Resultat Final

El servidor permet:

- Recepció de vídeo via RTMP
- Conversió automàtica a HLS
- Distribució web compatible amb navegadors moderns
- Reproducció en temps real des de qualsevol dispositiu compatible

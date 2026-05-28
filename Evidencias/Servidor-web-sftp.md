# 🖥️ 1. Pruebas del Servidor Web

El objetivo de estas pruebas es comprobar que el servidor web funciona correctamente, responde a peticiones externas y registra adecuadamente la actividad del sistema.

---

## ✅ 1.1 Comprobación del estado de NGINX

Se verificó que el servicio NGINX estuviera iniciado y funcionando correctamente.

### Comando utilizado

```bash
sudo systemctl status nginx
```

### Evidencia

<img width="928" height="352" alt="Captura de pantalla de 2026-05-28 12-09-23" src="https://github.com/user-attachments/assets/3b3f6e5c-81b2-452c-85e8-842fde4deb79" />

---

## ✅ 1.2 Acceso a la página web

Se comprobó que la página web estuviera accesible desde un navegador externo.

### Acceso realizado

```text
http://3.219.224.163
```

### Evidencia

<img width="1355" height="792" alt="Captura de pantalla de 2026-05-28 12-10-06" src="https://github.com/user-attachments/assets/ba5b84ef-8af3-4cb6-9d37-a82711963511" />

### Verificación de autenticación LDAP

En esta prueba se realizó correctamente el inicio de sesión utilizando un usuario autenticado mediante LDAP.

<img width="1676" height="831" alt="Captura de pantalla de 2026-05-28 12-13-57" src="https://github.com/user-attachments/assets/103b9522-4a3a-4e41-a549-52eca63796f1" />

---

## ✅ 1.3 Verificación de puertos abiertos

Se comprobó que los puertos necesarios para el servicio web estuvieran abiertos y escuchando conexiones.

### Comando utilizado

```bash
ss -tulnp
```

### Evidencia
<img width="909" height="263" alt="image" src="https://github.com/user-attachments/assets/54c52af7-d170-4a24-bdd9-e28d853f2d57" />


---

## ✅ 1.4 Verificación de logs del servidor web

Se comprobó que NGINX registrara correctamente las peticiones realizadas al servidor.

### Comandos utilizados

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/error.log
```

### Evidencias

<img width="936" height="398" alt="Captura de pantalla de 2026-05-28 12-20-05" src="https://github.com/user-attachments/assets/5a7dc09d-fa74-4744-ae92-87ff2a05de35" />

<img width="764" height="104" alt="Captura de pantalla de 2026-05-28 12-20-38" src="https://github.com/user-attachments/assets/15d31103-f63f-4b27-8b65-f253c6c6de56" />

---

# 🔐 2. Pruebas del Servidor SFTP

Las siguientes pruebas se realizaron para validar el funcionamiento seguro del servicio de transferencia de archivos mediante SFTP.

---

## ✅ 2.1 Comprobación del servicio SSH/SFTP

Se verificó que el servicio SSH estuviera activo, ya que SFTP funciona sobre este servicio.

### Comando utilizado

```bash
systemctl status ssh
```

### Evidencia

<img width="903" height="455" alt="Captura de pantalla de 2026-05-28 12-21-53" src="https://github.com/user-attachments/assets/41098015-0255-422f-9072-e701244b2473" />

---

## ✅ 2.2 Conexión mediante SFTP

Se realizó una conexión al servidor utilizando el protocolo SFTP.

### Comando utilizado

```bash
sftp usuario@3.219.224.163
```

### Verificación

Se comprobó que la autenticación fuera correcta y que el usuario pudiera acceder al servidor mediante SFTP.

### Evidencia

<img width="557" height="103" alt="Captura de pantalla de 2026-05-28 12-25-01" src="https://github.com/user-attachments/assets/cee902ba-58a7-416d-8bb5-0718e0cb9e4d" />

---

## ✅ 2.3 Descarga de archivos

Se verificó correctamente la descarga de archivos desde el servidor mediante SFTP.

### Comando utilizado

```bash
get archivo.txt
```

### Evidencia

<img width="878" height="84" alt="Captura de pantalla de 2026-05-28 12-43-50" src="https://github.com/user-attachments/assets/98530d21-4be9-4788-9be7-eccd0224e174" />

---

# 📊 Conclusión

Las pruebas realizadas permitieron verificar el correcto funcionamiento del servidor web y del servicio SFTP.

Se confirmó:

* Funcionamiento correcto de NGINX.
* Acceso web operativo desde navegador.
* Autenticación LDAP funcional.
* Puertos correctamente configurados.
* Registro correcto de logs.
* Conexión segura mediante SFTP.
* Transferencia correcta de archivos.

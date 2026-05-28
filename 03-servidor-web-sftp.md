# 03 - Servidor Web y SFTP

## Servidor Web (Nginx)

El servidor web se ha instalado utilizando **Nginx** sobre una instancia EC2 (`ip-10-0-1-237`), accesible públicamente en la IP `3.219.224.163`.

### Paso 1: Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### Paso 2: Verificación del servicio

```bash
sudo systemctl status nginx
```

### Paso 3: Comprobación desde el navegador

Accedemos a `http://3.219.224.163` desde el navegador y verificamos que nginx responde correctamente.

### Paso 4: Instalación de PHP para la página dinámica

```bash
sudo apt install php php-mysql php-fpm php-ldap -y
sudo systemctl restart nginx php8.3-fpm
```

### Paso 5: Configuración de Nginx para usar PHP

```bash
sudo nano /etc/nginx/sites-available/default
```

Configuración aplicada:

```nginx
server {
    listen 80;
    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }
}
```

```bash
sudo systemctl restart nginx php8.3-fpm
```

---

## Servidor SFTP

El servicio SFTP se ha configurado en el mismo servidor que el web (`ip-10-0-1-237`), autenticando los usuarios mediante **OpenLDAP**.

### Paso 1: Configuración del subsistema SFTP en SSH

```bash
sudo nano /etc/ssh/sshd_config
```

Se ha añadido la siguiente configuración al final del archivo:

```bash
Match Group sftpusers
    ChrootDirectory /home/%u
    ForceCommand internal-sftp
    PasswordAuthentication yes
    PubkeyAuthentication no
```

### Paso 2: Creación del grupo y directorio de uploads

```bash
sudo groupadd sftpusers
sudo mkdir -p /home/sftpuser1/uploads
sudo chown 2001:2001 /home/sftpuser1/uploads
```

### Paso 3: Comprobación de la conexión SFTP

```bash
sftp sftpuser1@localhost
```

Una vez dentro:

```bash
cd uploads
put /etc/hostname prueba.txt
ls
exit
```

---

## Integración SFTP + LDAP

Para que el servidor SFTP autentique los usuarios mediante LDAP, se ha configurado **nslcd** y **PAM** en el servidor web.

### Paso 1: Instalación de los paquetes necesarios

```bash
sudo apt install libnss-ldapd libpam-ldapd -y
```

Durante la instalación se ha configurado:
- **URI LDAP**: `ldap://10.0.1.209`
- **Base DN**: `dc=innovatetech,dc=local`

### Paso 2: Configuración de nslcd

```bash
sudo nano /etc/nslcd.conf
```

Configuración aplicada:

```bash
uri ldap://10.0.1.209/
base dc=innovatetech,dc=local
tls_reqcert never
```

### Paso 3: Configuración de nsswitch

```bash
cat /etc/nsswitch.conf | grep passwd
```

Debe mostrar:

```
passwd: files ldap
```

### Paso 4: Reinicio de los servicios

```bash
sudo systemctl restart nslcd ssh
```

### Paso 5: Verificación de que el sistema resuelve usuarios LDAP

```bash
getent passwd sftpuser1
```

Resultado esperado:

```
sftpuser1:x:2001:2001:sftpuser1:/home/sftpuser1:/bin/bash
```

El `uid=2001` confirma que el usuario proviene de **LDAP** y no del sistema local.

### Paso 6: Prueba de conexión SFTP autenticada vía LDAP

```bash
sftp sftpuser1@localhost
```

```bash
cd uploads
put /etc/hostname prueba_ldap.txt
ls
exit
```

---

## Página Web con autenticación LDAP y conexión a la BD

Se ha desarrollado una aplicación web en PHP que:
- Requiere **autenticación vía LDAP** para acceder
- Consulta la **base de datos MariaDB** en tiempo real
- Muestra todas las tablas de la BD de InnovateTech

### Paso 1: Creación del usuario de BD para la web

En la máquina Database-Server (`ip-10-0-1-208`):

```bash
mysql -u root -ppirineus -e "CREATE USER 'webuser'@'%' IDENTIFIED BY 'pirineus'; GRANT SELECT ON innovatetech.* TO 'webuser'@'%'; FLUSH PRIVILEGES;"
```

### Paso 2: Comprobación dinámica

Para verificar que la web es dinámica, modificamos un registro en la BD:

```bash
mysql -u root -ppirineus innovatetech -e "UPDATE Empleat SET telefon='111111111' WHERE dni='12345678A';"
```

Al recargar la página web, el cambio se refleja inmediatamente.

---

## Credenciales de acceso

| Servicio | URL / Dirección | Usuario | Contraseña |
|----------|----------------|---------|------------|
| Web | http://3.219.224.163 | sftpuser1 | Password123 |
| SFTP | sftp sftpuser1@3.219.224.163 | sftpuser1 | Password123 |

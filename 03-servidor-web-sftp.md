# 03 - Servidor Web i SFTP

## Servidor de Web (Nginx)

El servidor web s'ha instal·lat utilitzant **Nginx** sobre una instància EC2 (`ip-10-0-1-237`), accessible públicament a la IP `3.219.224.163`.

### Pas 1: Instal·lació de Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### Pas 2: Verificació del servei

```bash
sudo systemctl status nginx
```

### Pas 3: Comprovació des del navegador

Accedim a `http://3.219.224.163` des del navegador i verifiquem que nginx respon correctament.

### Pas 4: Instal·lació de PHP per la pàgina dinàmica

```bash
sudo apt install php php-mysql php-fpm php-ldap -y
sudo systemctl restart nginx php8.3-fpm
```

### Pas 5: Configuració de Nginx per usar PHP

```bash
sudo nano /etc/nginx/sites-available/default
```

Configuració aplicada:

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

El servei SFTP s'ha configurat al mateix servidor que el web (`ip-10-0-1-237`), autenticant els usuaris mitjançant **OpenLDAP**.

### Pas 1: Configuració del subsistema SFTP a SSH

```bash
sudo nano /etc/ssh/sshd_config
```

S'ha afegit la configuració següent al final del fitxer:

```bash
Match Group sftpusers
    ChrootDirectory /home/%u
    ForceCommand internal-sftp
    PasswordAuthentication yes
    PubkeyAuthentication no
```

### Pas 2: Creació del grup i directori d'uploads

```bash
sudo groupadd sftpusers
sudo mkdir -p /home/sftpuser1/uploads
sudo chown 2001:2001 /home/sftpuser1/uploads
```

### Pas 3: Comprovació de la connexió SFTP

```bash
sftp sftpuser1@localhost
```

Una vegada dins:

```bash
cd uploads
put /etc/hostname prueba.txt
ls
exit
```

---

## Integració SFTP + LDAP

Per tal que el servidor SFTP autentiqui els usuaris mitjançant LDAP, s'ha configurat **nslcd** i **PAM** al servidor web.

### Pas 1: Instal·lació dels paquets necessaris

```bash
sudo apt install libnss-ldapd libpam-ldapd -y
```

Durant la instal·lació s'ha configurat:
- **URI LDAP**: `ldap://10.0.1.209`
- **Base DN**: `dc=innovatetech,dc=local`

### Pas 2: Configuració de nslcd

```bash
sudo nano /etc/nslcd.conf
```

Configuració aplicada:

```bash
uri ldap://10.0.1.209/
base dc=innovatetech,dc=local
tls_reqcert never
```

### Pas 3: Configuració de nsswitch

```bash
cat /etc/nsswitch.conf | grep passwd
```

Ha de mostrar:

```
passwd: files ldap
```

### Pas 4: Reinici dels serveis

```bash
sudo systemctl restart nslcd ssh
```

### Pas 5: Verificació que el sistema resol usuaris LDAP

```bash
getent passwd sftpuser1
```

Resultat esperat:

```
sftpuser1:x:2001:2001:sftpuser1:/home/sftpuser1:/bin/bash
```

L'`uid=2001` confirma que l'usuari ve de **LDAP** i no del sistema local.

### Pas 6: Prova de connexió SFTP autenticada via LDAP

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

## Pàgina Web amb autenticació LDAP i connexió a la BD

S'ha desenvolupat una aplicació web en PHP que:
- Requereix **autenticació via LDAP** per accedir
- Consulta la **base de dades MariaDB** en temps real
- Mostra totes les taules de la BD d'InnovateTech

### Pas 1: Creació de l'usuari de BD per la web

A la màquina Database-Server (`ip-10-0-1-208`):

```bash
mysql -u root -ppirineus -e "CREATE USER 'webuser'@'%' IDENTIFIED BY 'pirineus'; GRANT SELECT ON innovatetech.* TO 'webuser'@'%'; FLUSH PRIVILEGES;"
```

### Pas 2: Comprovació dinàmica

Per verificar que la web és dinàmica, modifiquem un registre a la BD:

```bash
mysql -u root -ppirineus innovatetech -e "UPDATE Empleat SET telefon='111111111' WHERE dni='12345678A';"
```

En recarregar la pàgina web, el canvi es reflecteix immediatament.

---

## Credencials d'accés

| Servei | URL / Adreça | Usuari | Contrasenya |
|--------|-------------|--------|-------------|
| Web | http://3.219.224.163 | sftpuser1 | Password123 |
| SFTP | sftp sftpuser1@3.219.224.163 | sftpuser1 | Password123 |

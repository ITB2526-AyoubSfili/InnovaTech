# 04 - Servidor LDAP (OpenLDAP)

## Descripción

El directorio activo se ha instalado sobre una instancia EC2 separada (`ip-10-0-1-209`), accesible en la IP pública `54.85.232.177`.

- **Dominio**: `innovatetech.local`
- **Base DN**: `dc=innovatetech,dc=local`
- **URI**: `ldap://10.0.1.209`

---

## Instalación

### Paso 1: Instalación de OpenLDAP

```bash
sudo apt update
sudo apt install slapd ldap-utils -y
sudo dpkg-reconfigure slapd
```

Durante la configuración se han establecido los siguientes parámetros:
- **Dominio**: `innovatetech.local`
- **Organización**: `InnovateTech`
- **Contraseña admin**: configurada

### Paso 2: Verificación del servicio

```bash
sudo systemctl status slapd
```

### Paso 3: Comprobación de la estructura LDAP

```bash
sudo ldapsearch -x -H ldap://localhost -b dc=innovatetech,dc=local
```

---

## Creación de usuarios

### Paso 1: Creación del primer usuario (sftpuser1)

Creamos el archivo `usuari.ldif`:

```ldif
dn: uid=sftpuser1,ou=usuaris,dc=innovatetech,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: sftpuser1
sn: SFTP
givenName: User1
cn: sftpuser1
displayName: sftpuser1
uidNumber: 2001
gidNumber: 2001
loginShell: /bin/bash
homeDirectory: /home/sftpuser1
userPassword: Password123
```

```bash
ldapadd -x -D "cn=admin,dc=innovatetech,dc=local" -W -f usuari.ldif
```

### Paso 2: Creación del segundo usuario (sftpuser2)

Creamos el archivo `usuario2.ldif`:

```ldif
dn: uid=sftpuser2,ou=usuaris,dc=innovatetech,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: sftpuser2
sn: SFTP
givenName: User2
cn: sftpuser2
displayName: sftpuser2
uidNumber: 2002
gidNumber: 2001
loginShell: /bin/bash
homeDirectory: /home/sftpuser2
userPassword: Password123
```

```bash
ldapadd -x -D "cn=admin,dc=innovatetech,dc=local" -W -f usuario2.ldif
```

### Paso 3: Configuración de contraseñas

```bash
ldappasswd -x -D "cn=admin,dc=innovatetech,dc=local" -W -s Password123 "uid=sftpuser1,ou=usuaris,dc=innovatetech,dc=local"
ldappasswd -x -D "cn=admin,dc=innovatetech,dc=local" -W -s Password123 "uid=sftpuser2,ou=usuaris,dc=innovatetech,dc=local"
```

### Paso 4: Verificación de los usuarios creados

```bash
ldapsearch -x -b "dc=innovatetech,dc=local" "(objectClass=inetOrgPerson)"
```

Resultado esperado: 2 usuarios (`sftpuser1` y `sftpuser2`) con `uidNumber` 2001 y 2002 respectivamente.

---

## Credenciales de acceso

| Parámetro | Valor |
|-----------|-------|
| Servidor | ldap://10.0.1.209 |
| Base DN | dc=innovatetech,dc=local |
| Admin DN | cn=admin,dc=innovatetech,dc=local |
| Usuario 1 | sftpuser1 / Password123 |
| Usuario 2 | sftpuser2 / Password123 |

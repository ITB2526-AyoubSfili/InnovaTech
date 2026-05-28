# 04 - Servidor LDAP (OpenLDAP)

## Descripció

El directori actiu s'ha instal·lat sobre una instància EC2 separada (`ip-10-0-1-209`), accessible a la IP pública `54.85.232.177`.

- **Domini**: `innovatetech.local`
- **Base DN**: `dc=innovatetech,dc=local`
- **URI**: `ldap://10.0.1.209`

---

## Instal·lació

### Pas 1: Instal·lació d'OpenLDAP

```bash
sudo apt update
sudo apt install slapd ldap-utils -y
sudo dpkg-reconfigure slapd
```

Durant la configuració s'han establert els paràmetres següents:
- **Domini**: `innovatetech.local`
- **Organització**: `InnovateTech`
- **Contrasenya admin**: configurada

### Pas 2: Verificació del servei

```bash
sudo systemctl status slapd
```

### Pas 3: Comprovació de l'estructura LDAP

```bash
sudo ldapsearch -x -H ldap://localhost -b dc=innovatetech,dc=local
```

---

## Creació d'usuaris

### Pas 1: Creació del primer usuari (sftpuser1)

Creem el fitxer `usuari.ldif`:

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

### Pas 2: Creació del segon usuari (sftpuser2)

Creem el fitxer `usuario2.ldif`:

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

### Pas 3: Configuració de contrasenyes

```bash
ldappasswd -x -D "cn=admin,dc=innovatetech,dc=local" -W -s Password123 "uid=sftpuser1,ou=usuaris,dc=innovatetech,dc=local"
ldappasswd -x -D "cn=admin,dc=innovatetech,dc=local" -W -s Password123 "uid=sftpuser2,ou=usuaris,dc=innovatetech,dc=local"
```

### Pas 4: Verificació dels usuaris creats

```bash
ldapsearch -x -b "dc=innovatetech,dc=local" "(objectClass=inetOrgPerson)"
```

Resultat esperat: 2 usuaris (`sftpuser1` i `sftpuser2`) amb `uidNumber` 2001 i 2002 respectivament.

---

## Credencials d'accés

| Paràmetre | Valor |
|-----------|-------|
| Servidor | ldap://10.0.1.209 |
| Base DN | dc=innovatetech,dc=local |
| Admin DN | cn=admin,dc=innovatetech,dc=local |
| Usuari 1 | sftpuser1 / Password123 |
| Usuari 2 | sftpuser2 / Password123 |

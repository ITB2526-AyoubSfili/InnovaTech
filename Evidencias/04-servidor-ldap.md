# 🗂️ Servidor LDAP (OpenLDAP)

El objetivo de estas pruebas es comprobar que el servidor OpenLDAP funciona correctamente, que los usuarios están creados y que el directorio responde a consultas.

---

## ✅ 1. Comprobación del estado del servicio slapd

Se verificó que el servicio OpenLDAP estuviera iniciado y funcionando correctamente.

### Comando utilizado

```bash
sudo systemctl status slapd
```

### Evidencia

<img width="928" alt="Servicio slapd activo y corriendo desde 2026-05-21 06:30:47 UTC" src="img/ldap-slapd-status.png" />

---

## ✅ 2. Verificación de los usuarios creados

Se comprobó que los dos usuarios SFTP estuvieran correctamente registrados en el directorio LDAP con sus atributos POSIX.

### Comando utilizado

```bash
ldapsearch -x -b "dc=innovatetech,dc=local" "(objectClass=inetOrgPerson)"
```

### Evidencia

<img width="928" alt="ldapsearch mostrando sftpuser1 (uidNumber 2001) y sftpuser2 (uidNumber 2002) con todos sus atributos" src="img/ldap-usuarios.png" />

---

## ✅ 3. Comprobación de la estructura completa del directorio

Se verificó que el árbol de directorio estuviera creado correctamente con la organización, las unidades organizativas `ou=usuaris` y `ou=grups` y los usuarios.

### Comando utilizado

```bash
sudo ldapsearch -x -H ldap://localhost -b dc=innovatetech,dc=local
```

### Evidencia

<img width="928" alt="Estructura completa del directorio LDAP con dc=innovatetech,dc=local, ou=usuaris, ou=grups y sftpuser1" src="img/ldap-estructura.png" />

---

# 📊 Conclusión

Las pruebas realizadas permitieron verificar el correcto funcionamiento del servidor LDAP.

Se confirmó:

* Servicio slapd activo y funcionando.
* Usuarios `sftpuser1` y `sftpuser2` creados con atributos POSIX correctos.
* Estructura del directorio `dc=innovatetech,dc=local` creada correctamente.
* Directorio accesible y respondiendo consultas con éxito.

#  Servidor LDAP (OpenLDAP)

El objetivo de estas pruebas es comprobar que el servidor OpenLDAP funciona correctamente, que los usuarios están creados y que el directorio responde a consultas.

---

##  1. Comprobación del estado del servicio slapd

Se verificó que el servicio OpenLDAP estuviera iniciado y funcionando correctamente.

### Comando utilizado

```bash
sudo systemctl status slapd
```

### Evidencia

<img width="932" height="408" alt="ldap-slapd-status" src="https://github.com/user-attachments/assets/19699d68-0cf2-45fe-b77b-61c9392b94e0" />

---

##  2. Verificación de los usuarios creados

Se comprobó que los dos usuarios SFTP estuvieran correctamente registrados en el directorio LDAP con sus atributos POSIX.

### Comando utilizado

```bash
ldapsearch -x -b "dc=innovatetech,dc=local" "(objectClass=inetOrgPerson)"
```

### Evidencia

<img width="927" height="834" alt="ldap-usuarios" src="https://github.com/user-attachments/assets/9fee92c0-7f36-413b-b45a-6ec6c4b5e619" />

---

##  3. Comprobación de la estructura completa del directorio

Se verificó que el árbol de directorio estuviera creado correctamente con la organización, las unidades organizativas `ou=usuaris` y `ou=grups` y los usuarios.

### Comando utilizado

```bash
sudo ldapsearch -x -H ldap://localhost -b dc=innovatetech,dc=local
```

### Evidencia

<img width="861" height="890" alt="ldap-estructura" src="https://github.com/user-attachments/assets/357f3f2e-031a-4407-8a92-5aad5c97cc78" />

---

#  Conclusión

Las pruebas realizadas permitieron verificar el correcto funcionamiento del servidor LDAP.

Se confirmó:

* Servicio slapd activo y funcionando.
* Usuarios `sftpuser1` y `sftpuser2` creados con atributos POSIX correctos.
* Estructura del directorio `dc=innovatetech,dc=local` creada correctamente.
* Directorio accesible y respondiendo consultas con éxito.

# ⚙️ Automatización con Ansible — InnovateTech

El objetivo de estas pruebas es comprobar que Ansible puede conectarse a todos los servidores, ejecutar los playbooks correctamente y que los cambios se aplican en las 7 máquinas de forma simultánea.

---

## ✅ 1. Comprobación de conectividad con todos los servidores

Se verificó que Ansible pudiera conectarse a las 7 instancias EC2 mediante el módulo ping.

### Comando utilizado

```bash
sudo ansible servidors -m ping
```

### Evidencia

<img width="928" alt="Ansible ping SUCCESS en los 7 servidores" src="img/ansible-ping.png" />

---

## ✅ 2. Ejecución del playbook de usuario administrador

Se ejecutó el playbook que crea el usuario `adminitb` con permisos sudo y clave SSH en los 7 servidores simultáneamente.

### Comando utilizado

```bash
ansible-playbook /home/ubuntu/playbook_admin.yml
```

### Evidencia

<img width="928" alt="PLAY RECAP playbook_admin.yml ok=5 changed=4 en los 7 servidores" src="img/ansible-playbook-admin.png" />

---

## ✅ 3. Verificación del usuario adminitb en todas las máquinas

Se comprobó que el usuario `adminitb` se hubiera creado correctamente en todos los servidores y perteneciera al grupo sudo.

### Comando utilizado

```bash
ansible servidors -m command -a "id adminitb"
```

### Evidencia

<img width="928" alt="id adminitb en los 7 servidores mostrando uid=1001 groups=27(sudo)" src="img/ansible-id-adminitb.png" />

---

## ✅ 4. Ejecución del playbook de logs centralizados

Se ejecutó el playbook que configura el agente rsyslog en todos los servidores para enviar los logs al servidor central.

### Comando utilizado

```bash
ansible-playbook /home/ubuntu/playbook_rsyslog.yml
```

### Evidencia

<img width="928" alt="PLAY RECAP playbook_rsyslog.yml ok=4 changed=1 en los 7 servidores" src="img/ansible-playbook-rsyslog.png" />

---

## ✅ 5. Configuración de rsyslog en los clientes

Se verificó que el archivo `/etc/rsyslog.conf` de cada cliente tuviera la línea de redirección al servidor central `*.* @10.0.1.201:514`.

### Evidencia

<img width="928" alt="rsyslog.conf con la línea *.* @10.0.1.201:514 al final del archivo" src="img/ansible-rsyslog-conf.png" />

---

## ✅ 6. Logs recibidos en el servidor central

Se comprobó que el servidor de logs estuviera recibiendo correctamente los registros de todas las máquinas de la red.

### Comando utilizado

```bash
sudo ls /var/log/remote/ip-10-0-1-*/
```

### Evidencia

<img width="928" alt="Carpetas de logs por máquina: ip-10-0-1-201, 208, 209, 23, 237, 60" src="img/ansible-logs-remotos.png" />

---

# 📊 Conclusión

Las pruebas realizadas permitieron verificar el correcto funcionamiento de la automatización con Ansible.

Se confirmó:

* Conectividad exitosa con las 7 instancias EC2.
* Creación correcta del usuario `adminitb` con permisos sudo en todos los servidores.
* Configuración de logs centralizados aplicada correctamente en todas las máquinas.
* Logs de todos los servidores llegando al servidor central.

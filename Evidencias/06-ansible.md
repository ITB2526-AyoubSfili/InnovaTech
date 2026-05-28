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

<img width="929" height="938" alt="ansible-ping" src="https://github.com/user-attachments/assets/eadf5150-1c58-4798-b5c9-3ec19c0d096e" />


---

## ✅ 2. Ejecución del playbook de usuario administrador

Se ejecutó el playbook que crea el usuario `adminitb` con permisos sudo y clave SSH en los 7 servidores simultáneamente.

### Comando utilizado

```bash
ansible-playbook /home/ubuntu/playbook_admin.yml
```

### Evidencia

<img width="926" height="892" alt="ansible-playbook-admin" src="https://github.com/user-attachments/assets/9ac1e801-bf55-49f0-ab1c-2e9c60d389b5" />

---

## ✅ 3. Verificación del usuario adminitb en todas las máquinas

Se comprobó que el usuario `adminitb` se hubiera creado correctamente en todos los servidores y perteneciera al grupo sudo.

### Comando utilizado

```bash
ansible servidors -m command -a "id adminitb"
```

### Evidencia

<img width="592" height="248" alt="ansible-id-adminitb" src="https://github.com/user-attachments/assets/ea41de35-ae35-4adf-950b-5eace13eaac7" />

---

## ✅ 4. Ejecución del playbook de logs centralizados

Se ejecutó el playbook que configura el agente rsyslog en todos los servidores para enviar los logs al servidor central.

### Comando utilizado

```bash
ansible-playbook /home/ubuntu/playbook_rsyslog.yml
```

### Evidencia

<img width="925" height="762" alt="ansible-playbook-rsyslog" src="https://github.com/user-attachments/assets/8ca956c9-8535-4bdf-919f-561964e0c8f6" />

---

## ✅ 5. Configuración de rsyslog en los clientes

Se verificó que el archivo `/etc/rsyslog.conf` de cada cliente tuviera la línea de redirección al servidor central `*.* @10.0.1.201:514`.

### Evidencia

<img width="926" height="903" alt="ansible-rsyslog-conf" src="https://github.com/user-attachments/assets/f3b685fd-69ad-404e-acd7-f263c3db6658" />

---

## ✅ 6. Logs recibidos en el servidor central

Se comprobó que el servidor de logs estuviera recibiendo correctamente los registros de todas las máquinas de la red.

### Comando utilizado

```bash
sudo ls /var/log/remote/ip-10-0-1-*/
```

### Evidencia

<img width="716" height="363" alt="ansible-logs-remotos" src="https://github.com/user-attachments/assets/02aa49c4-595f-46b9-b4a3-08c815082703" />


---

# 📊 Conclusión

Las pruebas realizadas permitieron verificar el correcto funcionamiento de la automatización con Ansible.

Se confirmó:

* Conectividad exitosa con las 7 instancias EC2.
* Creación correcta del usuario `adminitb` con permisos sudo en todos los servidores.
* Configuración de logs centralizados aplicada correctamente en todas las máquinas.
* Logs de todos los servidores llegando al servidor central.

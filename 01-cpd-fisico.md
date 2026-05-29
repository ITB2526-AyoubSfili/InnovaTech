# PROYECTO TRANSVERSAL ASIXc1

# InnovateTech

# Diseño Físico e Infraestructura del CPD

---

# 1. Ubicación Física de la Sala

## 1.1 Decisión sobre la ubicación

A la hora de decidir dónde ubicar el CPD de InnovateTech, se estudiaron diversas opciones dentro del edificio corporativo.

Las opciones descartadas inicialmente fueron:

* El sótano, debido al riesgo de inundación y a la dificultad de evacuación en caso de emergencia.
* La última planta, por ser la zona más vulnerable desde el punto de vista estructural y la menos accesible para los servicios de emergencia.
* Las zonas próximas a paredes exteriores.
* Las ubicaciones situadas encima del aparcamiento.

Tras analizar todas las alternativas, se seleccionó la **segunda planta del edificio**, en una zona interior sin ventanas al exterior.

Esta ubicación ofrece:

* Mayor estabilidad térmica.
* Mayor seguridad física.
* Menor exposición a riesgos externos.
* Buena conexión con las vías de evacuación.

El suelo de esta zona soporta una carga de hasta **1.200 kg/m²**, suficiente para alojar racks completamente equipados.

### Criterios de selección

| Criterio              | Decisión                         | Motivo                                                |
| --------------------- | -------------------------------- | ----------------------------------------------------- |
| Planta                | Segunda planta interior          | Evita riesgos de inundación y problemas estructurales |
| Exposición exterior   | Zona interior sin ventanas       | Protección térmica y física                           |
| Zonas cercanas        | Alejada de salas comunes         | Menor tránsito de personas no autorizadas             |
| Servicios cercanos    | Sin tuberías de agua ni gas      | Reducción del riesgo de inundación y explosión        |
| Acceso a la sala      | Pasillo interno sin señalización | Dificulta la identificación de la sala                |
| Resistencia del suelo | ≥ 1.200 kg/m²                    | Soporte para racks de 42U completamente cargados      |

---

## 1.2 Climatización

Para la climatización de la sala se ha seleccionado un sistema **CRAC (Computer Room Air Conditioning)** de la gama **Liebert DS**.

Este sistema permite controlar simultáneamente:

* Temperatura.
* Humedad.
* Calidad del aire.

La sala utiliza una arquitectura de **pasillos fríos y calientes (Cold Aisle / Hot Aisle)** para mejorar la eficiencia energética.

El objetivo es mantener un valor de **PUE inferior a 1,4**.

### Parámetros de funcionamiento

| Parámetro        | Valor de trabajo | Límite crítico    | Sistema                             |
| ---------------- | ---------------- | ----------------- | ----------------------------------- |
| Temperatura      | 20–22 °C         | 18–24 °C          | CRAC Liebert DS (ASHRAE A2)         |
| Humedad relativa | 45–55 %          | 40–60 %           | Humidificador integrado             |
| Calidad del aire | ISO Clase 8      | < 10.000 part./m³ | Filtros HEPA H13                    |
| Flujo de aire    | Cold / Hot Aisle | Unidireccional    | Suelo técnico + extracción superior |
| Redundancia      | N+1              | —                 | 2 unidades CRAC                     |

### Figura 1 — Distribución 3D del CPD



<img width="658" height="440" alt="image" src="https://github.com/user-attachments/assets/ed5b10f0-f77f-430a-a316-5887541bfeb0" />

---

## 1.3 Medidas para evitar la identificación de la sala

Para reforzar la seguridad física se han aplicado las siguientes medidas:

* La puerta no dispone de ningún tipo de identificación.
* El acceso se realiza a través de un pasillo interno.
* El cableado permanece oculto dentro de las paredes.
* No existen ventanas visibles.
* Las entradas de aire del sistema CRAC no son visibles desde los pasillos.
* La sala no aparece en los planos generales accesibles al personal.

---

## 1.4 Cableado estructurado

El cableado sigue un sistema de identificación basado en:

* Código de colores.
* Etiquetado alfanumérico.

Formato utilizado:

```text
TIPO-ZONA-NÚMERO
```

Ejemplo:

```text
DA-CPD-07
```

### Especificaciones del cableado

| Tipo de cable       | Especificación               | Color    | Uso                                   |
| ------------------- | ---------------------------- | -------- | ------------------------------------- |
| Datos horizontales  | Cat6A U/FTP 10G              | Azul     | Conexiones entre racks y patch panels |
| Uplinks entre racks | Fibra OM4 LC-LC              | Amarillo | Conexiones 10G                        |
| Eléctrico           | 3×2,5 mm² con toma de tierra | Rojo     | Alimentación eléctrica                |
| Gestión/IPMI        | Cat6 UTP                     | Verde    | Administración de servidores          |
| Bandejas            | Metálicas de 100×60 mm       | —        | Separación física del cableado        |

### Figura 2 — Distribución del cableado


<img width="658" height="440" alt="image" src="https://github.com/user-attachments/assets/707ad612-46b5-459f-851e-24f8f7854eea" />


---

## 1.5 Suelo técnico y falso techo

### Suelo técnico

* Elevación: 40 cm.
* Paneles de acero galvanizado de 60×60 cm.
* Resistencia: 1.200 kg/m².
* Distribución del aire frío.
* Paso del cableado de datos.

### Falso techo técnico

* Altura: 3 metros.
* Cableado eléctrico.
* Backbone de red.
* Sensores y sistemas de detección de incendios.

Esta separación facilita el mantenimiento y reduce las interferencias.

---

# 2. Infraestructura IT

## 2.1 Servidores (Instancias EC2)

Se ha adoptado una arquitectura basada en AWS.

Todas las máquinas han sido configuradas manualmente.

### Infraestructura desplegada

| Instancia    | Tipo     | Servicio               |
| ------------ | -------- | ---------------------- |
| ec2-web-sftp | t2.micro | Web + SFTP             |
| ec2-ldap     | t2.micro | OpenLDAP               |
| ec2-logs     | t2.micro | Centralización de logs |
| ec2-bbdd     | t2.micro | MariaDB                |
| ec2-audio    | t2.micro | Icecast2               |
| ec2-video    | t2.micro | NGINX-RTMP             |
| ec2-jitsi    | t2.micro | Jitsi Meet             |

---

## 2.2 Patch Panels y Switches

| Elemento             | Modelo                 | Unidades | Ubicación       |
| -------------------- | ---------------------- | -------- | --------------- |
| Patch panel de datos | 24 puertos Cat6A       | 1        | Rack 1          |
| Patch panel de red   | 48 puertos Cat6A       | 1        | Rack 2          |
| Switch Core          | Cisco SG350X 10G       | 1        | Rack 2          |
| Switch de acceso     | Cisco SG350 24p        | 2        | Rack 1 y Rack 2 |
| Firewall             | Netgate 2100 + pfSense | 1        | Rack 2          |
| Consola KVM LCD      | Consola Rack 1U        | 1        | Rack 2          |

---

## 2.3 Distribución de los Racks

### Rack 1 — Servidores

| Unidad  | Equipo           |
| ------- | ---------------- |
| U1-U2   | Patch Panel      |
| U3      | Switch de acceso |
| U4-U5   | EC2 Web + SFTP   |
| U6-U7   | EC2 LDAP         |
| U8-U9   | EC2 Logs         |
| U10-U11 | EC2 BBDD         |
| U12-U13 | EC2 Audio        |
| U14-U16 | EC2 Video        |
| U17-U19 | EC2 Jitsi        |
| U20-U42 | Espacio libre    |

### Rack 2 — Red y Alimentación

| Unidad  | Equipo           |
| ------- | ---------------- |
| U1-U2   | Patch Panel      |
| U4-U5   | Switch Core      |
| U6      | Switch de acceso |
| U8-U9   | Firewall         |
| U10-U11 | KVM              |
| U14-U16 | SAI APC          |
| U17-U19 | SAI APC          |
| U20-U42 | Espacio libre    |

### Figura 3 — Distribución de los Racks


<img width="655" height="431" alt="image" src="https://github.com/user-attachments/assets/d3d224ca-6e9a-428c-809c-0bc7555b1949" />


---

# 3. Infraestructura Eléctrica

## 3.1 Alimentación Redundante

Todos los servidores disponen de:

* Fuentes de alimentación dobles (PSU Dual).
* Conexión a dos cuadros eléctricos independientes.
* Tolerancia a variaciones de tensión de hasta el 5%.

Características:

* Cuadro A independiente.
* Cuadro B independiente.
* Diferenciales independientes.
* Magnetotérmicos independientes.

Esta configuración garantiza una alta disponibilidad del servicio.

---

## 3.2 SAI — Cálculo de Autonomía

El consumo total estimado del CPD es de **1.130 W**.

Para garantizar la continuidad operativa se han instalado:

* 2 × APC Smart-UPS RT 3000VA.
* Tecnología Online (Doble Conversión).

#  4. Seguridad Física y Lógica

---

## 4.1 Seguridad Física

La seguridad física de la sala se ha diseñado en múltiples capas. El objetivo principal es impedir el acceso de personas no autorizadas y, en caso de acceso indebido, minimizar cualquier posible daño o exfiltración de información.

A continuación, se detallan los elementos de seguridad implementados:

| Elemento                   | Descripción                                                                                                                                                                                                                                                                                                                  |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Control de acceso**      | Lector biométrico de huella dactilar (**Suprema BioEntry W2**) combinado con tarjeta RFID como segundo factor de autenticación. Cada entrada y salida queda registrada con fecha, hora, usuario y duración de la visita. El acceso está restringido exclusivamente al personal de TI autorizado.                             |
| **Videovigilancia**        | 4 cámaras IP PTZ (**Axis P3245-V, 4MP**) instaladas en las esquinas de la sala, además de una cámara adicional en el pasillo de acceso. Grabación continua 24/7 con retención de 30 días en un NVR local cifrado. Monitorización en tiempo real desde la consola de seguridad.                                               |
| **Detección de incendios** | Sistema **VESDA (Very Early Smoke Detection Apparatus)** que analiza continuamente el aire del falso techo para detectar partículas de humo antes de que sean visibles. También se han instalado sensores térmicos cada 3 metros. El panel de alarma es independiente y envía alertas automáticas por SMS al personal de TI. |
| **Extinción de incendios** | Sistema de extinción mediante gas **Novec 1230 (3M)**, un agente limpio que no deja residuos ni daña los equipos electrónicos. La activación puede realizarse automáticamente mediante sensores o manualmente mediante pulsador de emergencia. El tiempo de descarga es inferior a 10 segundos.                              |
| **Evacuación**             | Puerta principal con control de acceso y puerta de emergencia en el lateral norte con barra antipánico. Señalización LED autónoma con batería de respaldo de 3 horas. Pasillos de 90 cm entre racks para facilitar la evacuación.                                                                                            |
| **Estructura de la sala**  | Puerta blindada de acero clase **RC3** con marco reforzado. Paredes de hormigón armado de 20 cm. Sala completamente cerrada, sin ventanas ni aperturas exteriores. Sensores de inundación instalados bajo el suelo técnico con alertas automáticas.                                                                          |

---

##  4.2 Seguridad Lógica

La seguridad lógica complementa la seguridad física. No basta con proteger el acceso a la sala si los sistemas informáticos no están correctamente securizados.

Por este motivo, se han implementado diferentes capas de protección orientadas a garantizar la confidencialidad, integridad y disponibilidad de la información.

| Elemento                        | Descripción                                                                                                                                                                                                                                                                                                |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Autenticación**               | Todos los servicios se autentican mediante **OpenLDAP (ec2-ldap)**. Los administradores utilizan autenticación de doble factor (**TOTP**) mediante Google Authenticator. El servicio SFTP se autentica contra LDAP utilizando PAM. El acceso directo con el usuario `root` está deshabilitado.             |
| **Acceso SSH**                  | El acceso a todas las máquinas se realiza exclusivamente mediante claves pública/privada. No se permiten contraseñas SSH. Cada usuario dispone únicamente de los permisos necesarios para desempeñar su trabajo. Todas las conexiones SSH quedan registradas en Graylog.                                   |
| **Firewall**                    | Firewall **pfSense** desplegado sobre **Netgate 2100** con política por defecto de bloqueo total (*deny all*). Las reglas se aplican por capas: perímetro (Internet ↔ CPD) y red interna (entre instancias EC2). El sistema IDS/IPS **Snort** analiza el tráfico en tiempo real.                           |
| **Monitorización**              | **rsyslog** centraliza los registros de todas las instancias EC2 mediante Syslog. Cuando se detecta un evento crítico (intento de acceso no autorizado, caída de servicios o uso elevado de recursos) se generan alertas automáticas por correo electrónico y Telegram.                       |
| **Copias de seguridad**         | Copias de seguridad diarias cifradas mediante **AES-256** almacenadas en un bucket privado de **Amazon S3**. Retención de 30 días. Cada mes se verifica la integridad de las copias utilizando checksums **SHA-256**. Además, se realiza una copia semanal en **S3 Glacier** para archivado a largo plazo. |
| **RAID**                        | Configuración **RAID 1** para los discos del sistema operativo, proporcionando tolerancia al fallo de un disco. Configuración **RAID 5** para los datos de logs y bases de datos. Cada servidor dispone de un disco hot-swap de repuesto para recuperación inmediata.                                      |
| **Gestión de vulnerabilidades** | Las actualizaciones de seguridad se aplican automáticamente mediante **unattended-upgrades** en todas las instancias. Se realiza un análisis mensual de vulnerabilidades con **OpenVAS** y una revisión trimestral de permisos y roles LDAP.                                                               |

# 5 Prevención de Riesgos Laborales (PRL)

La sala del CPD presenta condiciones especiales de trabajo debido al ruido de los sistemas de refrigeración, la temperatura controlada y la presencia de equipos eléctricos de alta tensión. Por ello, se han definido las siguientes medidas de seguridad:

* Iluminación de emergencia autónoma con batería de 3 horas, activación automática y un mínimo de 50 lux en pasillos.

* Pasillos de al menos 90 cm entre racks para facilitar la evacuación y el mantenimiento.

* Señalización normalizada ISO 7010 en zonas de riesgo eléctrico y alta tensión.

* Formación anual obligatoria para el personal de TI sobre primeros auxilios, evacuación y manipulación segura de equipos eléctricos.

* Prohibición de comer, beber o introducir líquidos dentro de la sala.

* Equipos de protección individual (EPI) disponibles en la entrada:

  * Guantes aislantes clase 0 (hasta 1.000 V).
  * Gafas de protección.
  * Calzado antiestático.

* Protocolo de trabajo en caliente:

  * Permiso escrito obligatorio.
  * Presencia de un acompañante.
  * Notificación previa al responsable de TI.

* Tiempo máximo de permanencia continua de 30 minutos debido al ruido de los sistemas de ventilación.

* Extintor manual de CO₂ disponible junto a la salida para incendios eléctricos de pequeña magnitud.


### Consumo estimado

| Elemento                    | Consumo     |
| --------------------------- | ----------- |
| 7 instancias EC2            | 700 W       |
| 2 Switches + Firewall       | 180 W       |
| 2 SAI (consumo propio)      | 120 W       |
| Iluminación y otros equipos | 130 W       |
| **TOTAL**                   | **1.130 W** |

### Resultados

* Potencia disponible: 5.400 W.
* Carga real: 19 %.
* Autonomía estimada: 2 horas.
* Tiempo suficiente para activar el generador diésel de respaldo.

---

# Conclusiones

La infraestructura del CPD de InnovateTech ha sido diseñada siguiendo criterios profesionales de disponibilidad, seguridad y escalabilidad.

Los principales puntos fuertes son:

* Ubicación física segura.
* Climatización profesional mediante CRAC.
* Arquitectura de pasillos fríos y calientes.
* Cableado estructurado correctamente documentado.
* Alimentación eléctrica redundante.
* SAI online de doble conversión.
* Infraestructura cloud basada en AWS.
* Espacio disponible para futuras ampliaciones.
* Cumplimiento de las buenas prácticas en el diseño de centros de procesamiento de datos.

Este diseño garantiza la continuidad de los servicios y permite el crecimiento futuro de la infraestructura sin necesidad de rediseñar el CPD.

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

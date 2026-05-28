#  1. Infraestructura de Red y Arquitectura Cloud (AWS)

La arquitectura cloud de **InnovateTech** se basa en un entorno seguro, escalable y altamente disponible desplegado sobre **Amazon Web Services (AWS)**. A continuación, se detalla la configuración de la infraestructura utilizando las evidencias obtenidas desde el panel de administración de AWS.

---

#  1.1 Gestión de Servidores: Instancias EC2

En este apartado se muestra el estado operativo y la configuración de red de los servidores virtuales desplegados en AWS.

## Evidencia

> <img width="1561" height="825" alt="Captura de pantalla de 2026-05-27 10-21-48" src="https://github.com/user-attachments/assets/c2a4a108-d53f-4b56-be1d-0bd764df6ad6" />


*(Esta captura muestra el listado completo de instancias EC2 desplegadas, incluyendo Proxy, MariaDB, Rsyslog y otros servicios del proyecto).*

## Explicación

En esta vista del panel de **Amazon EC2** se puede verificar que todas las instancias críticas del proyecto se encuentran en estado **Running**, indicando que los servicios están operativos y disponibles.

Se observan las direcciones **IP privadas** asignadas a cada instancia dentro del rango **10.0.1.x**, utilizadas para la comunicación interna entre servidores sin necesidad de exponerlos directamente a Internet.

Asimismo, cada instancia se encuentra asociada a una **Zona de Disponibilidad (Availability Zone)** específica, proporcionando mayor estabilidad y tolerancia ante posibles fallos de infraestructura.

### Servicios desplegados

| Instancia  | Función                                   |
| ---------- | ----------------------------------------- |
| Proxy      | Punto de acceso principal a los servicios |
| MariaDB    | Base de datos principal                   |
| Rsyslog    | Centralización de registros               |
| LDAP       | Servicio de autenticación                 |
| Jitsi      | Plataforma de videoconferencias           |
| Icecast    | Servicio de audio en streaming            |
| NGINX-RTMP | Servicio de vídeo en streaming            |

---

#  1.2 Seguridad Perimetral: Security Groups

Para garantizar la protección de la infraestructura se han configurado diferentes **Security Groups**, que actúan como cortafuegos virtuales controlando el tráfico entrante y saliente de cada instancia.

## Evidencia

> <img width="1533" height="277" alt="Captura de pantalla de 2026-05-27 10-22-43" src="https://github.com/user-attachments/assets/8a7c2d01-1c6e-40fa-a55e-c82ca0f1a724" />


*(Esta captura muestra la configuración detallada de un Security Group asociado a los servicios principales).*

## Explicación

En la captura se muestran las **Inbound Rules (Reglas de Entrada)** definidas para permitir únicamente el tráfico necesario para el funcionamiento de los servicios.

Se han habilitado puertos específicos como:

| Puerto | Protocolo | Servicio                                |
| ------ | --------- | --------------------------------------- |
| 22     | TCP       | SSH (Administración remota)             |
| 80     | TCP       | HTTP                                    |
| 443    | TCP       | HTTPS                                   |
| 3306   | TCP       | MariaDB                                 |
| 5060   | UDP/TCP   | Servicios de comunicación               |
| 8000   | TCP       | Servicios de audio                      |
| 8080   | TCP       | Servicios de streaming                  |
| 8096   | TCP       | Servicios de streaming                  |
| 10000 | UDP       | Tráfico multimedia de videoconferencias |


### Medidas de Seguridad

* Restricción del acceso SSH únicamente a IPs autorizadas.
* Acceso a MariaDB limitado a servidores internos.
* Bloqueo de conexiones no autorizadas.
* Aplicación del principio de mínimo privilegio.
* Segmentación de servicios mediante grupos de seguridad independientes.

Gracias a esta configuración, la base de datos y los servicios internos permanecen protegidos frente a accesos externos no autorizados.

---

#  1.3 Conectividad y Acceso a Internet: Internet Gateway

El acceso controlado a Internet se realiza mediante un **Internet Gateway (IGW)** asociado a la VPC principal.

## Evidencia

> <img width="1564" height="715" alt="Captura de pantalla de 2026-05-27 10-24-08" src="https://github.com/user-attachments/assets/d4d6a8fc-41dc-4715-a72a-013c555c7452" />


*(Esta captura muestra el Internet Gateway asociado a la VPC con estado "Attached").*

## Explicación

El **Internet Gateway** es el componente responsable de conectar la red privada virtual (**VPC**) con Internet.

En la captura se puede observar que el gateway se encuentra en estado **Attached**, indicando que está correctamente vinculado a la VPC principal del proyecto.

### Funciones principales

* Permitir el acceso a Internet de las instancias públicas.
* Facilitar la recepción de conexiones externas.
* Permitir actualizaciones del sistema operativo.
* Habilitar la comunicación con servicios externos.

Sin este componente, el servidor Proxy no podría recibir las peticiones procedentes de los clientes externos ni redirigirlas hacia los servicios internos.

---

#  1.4 Visualización de la Red: Mapa de Recursos VPC

Para disponer de una visión global de la infraestructura se utiliza el mapa de recursos proporcionado por AWS.

## Evidencia

> <img width="1387" height="375" alt="image (1)" src="https://github.com/user-attachments/assets/7ae3dcc7-5a1f-4df8-81d9-95e9f23f9a47" />


*(Esta captura muestra el diagrama lógico de la VPC y sus recursos asociados).*

## Explicación

El diagrama representa visualmente la arquitectura de red desplegada para InnovateTech.

### Componentes principales

#### VPC

La **Virtual Private Cloud (VPC)** constituye el contenedor principal de toda la infraestructura.

```text
10.0.0.0/16
```

Dentro de ella se encuentran organizados todos los recursos de red.

#### Subnets

Las subredes permiten dividir la infraestructura en distintos segmentos:

* Subredes públicas.
* Subredes privadas.
* Segmentación por servicios.

Esta división mejora tanto la seguridad como la organización de la red.

#### Route Tables

Las tablas de rutas determinan cómo circula el tráfico entre:

* Subredes internas.
* Internet Gateway.
* Recursos externos.

Gracias a estas reglas se puede definir qué servidores tienen acceso a Internet y cuáles permanecen completamente aislados.

### Beneficios de esta arquitectura

* Segmentación lógica de la red.
* Mayor seguridad.
* Facilidad de administración.
* Escalabilidad futura.
* Reducción de la superficie de ataque.

---

#  1.5 Resumen de Configuración

| Captura             | Apartado             | Propósito                                                     |
| ------------------- | -------------------- | ------------------------------------------------------------- |
| 2026-05-27 10-21-48 | 1.1 Instancias EC2   | Mostrar los servidores desplegados y sus direcciones IP       |
| 2026-05-27 10-22-43 | 1.2 Security Groups  | Demostrar la configuración de seguridad y filtrado de puertos |
| 2026-05-27 10-24-08 | 1.3 Internet Gateway | Verificar la conectividad con Internet                        |
| image (1).png       | 1.4 Arquitectura VPC | Mostrar la estructura lógica completa de la red               |

---

#  Conclusiones

La infraestructura cloud desplegada en AWS proporciona una plataforma robusta, segura y escalable para los servicios de InnovateTech.

Los elementos clave de esta arquitectura son:

* Uso de instancias EC2 para la ejecución de servicios.
* Segmentación de la red mediante VPC y Subnets.
* Protección perimetral mediante Security Groups.
* Acceso controlado a Internet a través de Internet Gateway.
* Comunicación interna mediante direcciones IP privadas.
* Arquitectura preparada para futuras ampliaciones y crecimiento de la organización.

Esta configuración garantiza la disponibilidad de los servicios, la protección de los datos y una administración eficiente de los recursos desplegados en la nube.


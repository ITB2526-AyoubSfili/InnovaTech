# 1.1. Diseño Entidad-Relación y Modelo Relacional**

¿Qué es un diagrama E/R y para qué sirve?

En un diagrama Entidad-Relación es una representación visual de cómo se organizan los datos de un sistema antes de escribir ningún código. Sirve para planificar qué información se va a guardar y cómo se relacionan las diferentes partes entre sí.
En este proyecto, nosotros necesitábamos diseñar la base de datos de InnovateTech, una empresa con empleados, departamentos, un sistema de videoconferencias y un catálogo de vídeos en streaming. Antes de crear ninguna tabla en MySQL, nosotros debemos tener claro el modelo de datos.

## Relaciones y Cardinalidades

Las relaciones indican cómo se conectan las entidades entre sí. La cardinalidad especifica cuántos elementos de una entidad pueden relacionarse con cuántos de la otra. La siguiente tabla resume todas las relaciones del modelo implementado:

# 11. Diseño de la Base de Datos

A continuación, se detalla la estructura lógica y el diseño de las entidades que componen el sistema de Innovate Tech.

### Diagramas del Modelo
Puedes consultar el diagrama completo en el siguiente enlace:
[Diagrama del Modelo Relacional - Google Drive](https://drive.google.com/file/d/1BxvITU8HYOXiMdQY0S00P1nkJsLVQZRA/view)

### Identificación de las entidades

* **DEPARTAMENT.** Representa los departamentos internos de la empresa (ventas, soporte técnico, administración y logística). Cada empleado pertenece obligatoriamente a uno.
* **EMPLEAT.** Representa a cada trabajador de la empresa. El identificador único es el DNI, que no puede repetirse ni estar vacío.
* **USUARI_COMUNICACIO.** No todos los empleados participan en el sistema de videoconferencias. Esta entidad representa únicamente a aquellos dados de alta en el sistema de llamadas. Incluye el estado del usuario (activo o bloqueado), que será utilizado por los triggers de seguridad.
* **GRUP_QUALITAT.** Define los niveles de calidad de streaming disponibles (alta, media, baja). Cada grupo establece el máximo de minutos mensuales y el máximo de llamadas diarias permitidas a sus usuarios. Esta entidad es la base del control de cuotas.
* **TRUCADA.** Registra cada llamada realizada en el sistema. Almacena quién la inició, quién la recibió, cuándo empezó, cuándo terminó, la duración total, la calidad de servicio utilizada y la valoración opcional del usuario.
* **VIDEO.** Contiene el catálogo de vídeos disponibles en el servidor de streaming. Para cada vídeo se guarda el título, la descripción, la categoría, la duración, la fecha de publicación y el enlace directo al servidor.
* **MESURA_BW.** Almacena los resultados de las pruebas de ancho de banda realizadas por los operarios. Incluye la velocidad de descarga, la de subida, la latencia, el resultado (aceptable o no aceptable) y observaciones adicionales.
* **AVIS_AUDITORIA.** Es la tabla donde los triggers de seguridad escriben automáticamente cuando detectan un intento de acceso no autorizado. Registra el usuario de la base de datos, la tabla afectada, la operación intentada y la fecha y hora exactas.
* **CONTROL_BACKUP.** Registra cada ejecución del evento automático de copia de seguridad. Guarda la fecha, las tablas incluidas y el resultado de la operación.

| Relación | Cardinalidad | Descripción |
| :--- | :---: | :--- |
| **Departament -> Empleat** | 1:N | Un departamento tiene muchos empleados. Cada empleado pertenece a un único departamento. |
| **Empleat -> Usuari_comunicació** | 1:0..1 | Un empleado puede tener como máximo un usuario de comunicación, o ninguno. |
| **Usuari_comunicació -> Grup_qualitat** | N:1 | Muchos usuarios pertenecen al mismo grupo de calidad de streaming. |
| **Usuari_comunicació -> Trucada (origina)** | 1:N | Un usuario puede originar muchas llamadas a lo largo del tiempo. |
| **Usuari_comunicació -> Trucada (recibe)** | 1:N | Un usuario puede recibir muchas llamadas a lo largo del tiempo. |
| **Grup_qualitat -> Trucada** | 1:N | Cada llamada registra la configuración de calidad con la que fue realizada. |
| **Usuari_comunicació -> Mesura_BW** | 1:N | Un operario puede realizar múltiples mediciones de ancho de banda. |

Las entidades VIDEO, AVIS_AUDITORIA y CONTROL_BACKUP son independientes. Los triggers escriben directamente en AVIS_AUDITORIA sin necesitar una relación explícita, y VIDEO es un catálogo que se alimenta de forma independiente desde el servidor de streaming

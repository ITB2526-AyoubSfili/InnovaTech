# 3.2. Implementación SQL y Automatización (Triggers / Backups)

---

## Instalación del servicio de bases de datos

El primer paso fue instalar el servidor de bases de datos **MariaDB** en la máquina dedicada (`ip-10-0-1-208`). MariaDB es una bifurcación comunitaria de MySQL, ampliamente utilizada en entornos de producción por su rendimiento y compatibilidad.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install mariadb-server -y
```

Una vez instalado, habilitamos y arrancamos el servicio para que se inicie automáticamente con el sistema:

```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
sudo systemctl status mariadb
```

---

## Configuración de la Fortificación del Servidor (MariaDB)

Una vez instalado MariaDB, es imprescindible **blindar el servidor** antes de cualquier uso en producción. Para ello se ejecuta el script oficial de seguridad:

```bash
sudo mysql_secure_installation
```

### Acciones clave realizadas durante el proceso:

| Acción | Descripción |
|--------|-------------|
| **Blindaje de Root** | Configuración de contraseña robusta y activación del método de autenticación `unix_socket` para evitar accesos no autorizados. |
| **Limpieza de Seguridad** | Eliminación completa de usuarios anónimos y de la base de datos `test`, que vienen por defecto y representan un riesgo. |
| **Acceso Remoto** | Habilitado temporalmente para facilitar el desarrollo. En producción se restringirá exclusivamente a `localhost`. |
| **Actualización de Privilegios** | Aplicación inmediata de todos los cambios mediante `FLUSH PRIVILEGES` para que surtan efecto sin reiniciar el servicio. |

### Verificación del estado tras la configuración:

```bash
sudo mysql -u root -p
```

```sql
-- Verificar que no existen usuarios anónimos
SELECT User, Host FROM mysql.user;

-- Verificar que la base de datos test ha sido eliminada
SHOW DATABASES;

-- Refrescar privilegios manualmente si fuera necesario
FLUSH PRIVILEGES;
```

---

## 🗄️ Implementación de la Base de Datos InnovateTech

> 🔗 **Código fuente completo:** [Ver en Google Drive](https://drive.google.com/file/d/1UDlYD38ebCAn6BMpwJ99NEg2WSlSFCdf/view)

La base de datos `innovatetech` fue diseñada e implementada para dar soporte a toda la infraestructura de la empresa: gestión de empleados, sistema de videoconferencias, catálogo de vídeos en streaming y auditoría de seguridad.

### Creación de la base de datos

```sql
CREATE DATABASE IF NOT EXISTS innovatetech
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE innovatetech;
```

---

### Estructura Organizativa

Se crearon las tablas base de la organización con **integridad referencial mediante claves foráneas**, garantizando que los datos siempre estén relacionados de forma coherente.

```sql
-- Tabla de Departamentos
CREATE TABLE Departament (
  codi_dep    INT           NOT NULL AUTO_INCREMENT,
  nom         VARCHAR(100)  NOT NULL,
  telefon     VARCHAR(20),
  PRIMARY KEY (codi_dep)
);

-- Tabla de Empleados
CREATE TABLE Empleat (
  dni         VARCHAR(9)    NOT NULL,
  nom         VARCHAR(50)   NOT NULL,
  cognoms     VARCHAR(100)  NOT NULL,
  adreca      VARCHAR(200),
  telefon     VARCHAR(20),
  codi_dep    INT           NOT NULL,
  PRIMARY KEY (dni),
  CONSTRAINT fk_emp_dep
    FOREIGN KEY (codi_dep) REFERENCES Departament(codi_dep)
    ON UPDATE CASCADE ON DELETE RESTRICT
);
```

> **ON UPDATE CASCADE**: Si cambia el código de un departamento, se actualiza automáticamente en todos los empleados.  
> **ON DELETE RESTRICT**: No se puede eliminar un departamento que tenga empleados asignados.

---

### Gestión de Comunicaciones (QoS)

Se implementaron tablas para controlar el acceso al sistema de videoconferencias con **perfiles de calidad de servicio (QoS)**, limitando los minutos mensuales y las llamadas diarias por usuario.

```sql
-- Grupos de calidad de servicio
CREATE TABLE Grup_qualitat (
  id_grup          INT           NOT NULL AUTO_INCREMENT,
  nom              VARCHAR(50)   NOT NULL,
  qualitat         ENUM('alta','mitja','baixa') NOT NULL,
  max_minuts_mes   INT           NOT NULL DEFAULT 600,
  max_trucades_dia INT           NOT NULL DEFAULT 20,
  PRIMARY KEY (id_grup)
);

-- Usuarios del sistema de comunicación
CREATE TABLE Usuari_comunicacio (
  id_usuari   INT           NOT NULL AUTO_INCREMENT,
  dni_empleat VARCHAR(9)    NOT NULL,
  email       VARCHAR(100)  NOT NULL UNIQUE,
  extensio    VARCHAR(10),
  estat       ENUM('actiu','bloquejat') NOT NULL DEFAULT 'actiu',
  tipus       ENUM('intern','extern')   NOT NULL DEFAULT 'intern',
  id_grup     INT           NOT NULL,
  PRIMARY KEY (id_usuari),
  CONSTRAINT fk_usuari_empleat
    FOREIGN KEY (dni_empleat) REFERENCES Empleat(dni)
    ON UPDATE CASCADE ON DELETE RESTRICT,
  CONSTRAINT fk_usuari_grup
    FOREIGN KEY (id_grup) REFERENCES Grup_qualitat(id_grup)
    ON UPDATE CASCADE ON DELETE RESTRICT
);
```

---

### Registro de Actividad

Tablas para el historial completo de llamadas y el catálogo de vídeos del servidor de streaming:

```sql
-- Registro de llamadas
CREATE TABLE Trucada (
  id_trucada      INT       NOT NULL AUTO_INCREMENT,
  id_originador   INT       NOT NULL,
  id_destinatari  INT       NOT NULL,
  inici           DATETIME  NOT NULL,
  fi              DATETIME,
  durada_total    INT       COMMENT 'Duración en segundos',
  id_grup         INT       NOT NULL,
  puntuacio       TINYINT   CHECK (puntuacio BETWEEN 1 AND 5),
  comentari       VARCHAR(300),
  PRIMARY KEY (id_trucada),
  CONSTRAINT fk_trucada_orig
    FOREIGN KEY (id_originador) REFERENCES Usuari_comunicacio(id_usuari)
    ON UPDATE CASCADE ON DELETE RESTRICT,
  CONSTRAINT fk_trucada_dest
    FOREIGN KEY (id_destinatari) REFERENCES Usuari_comunicacio(id_usuari)
    ON UPDATE CASCADE ON DELETE RESTRICT,
  CONSTRAINT fk_trucada_grup
    FOREIGN KEY (id_grup) REFERENCES Grup_qualitat(id_grup)
    ON UPDATE CASCADE ON DELETE RESTRICT
);

-- Catálogo de vídeos en streaming
CREATE TABLE Video (
  id_video        INT           NOT NULL AUTO_INCREMENT,
  titol           VARCHAR(150)  NOT NULL,
  descripcio      TEXT,
  categoria       VARCHAR(50),
  durada          INT           COMMENT 'Duración en segundos',
  data_publicacio DATE,
  url_streaming   VARCHAR(300)  NOT NULL,
  PRIMARY KEY (id_video)
);
```

---

### Mantenimiento y Auditoría

Tablas para el monitoreo continuo del ancho de banda y el registro automático de copias de seguridad y eventos de auditoría:

```sql
-- Mediciones de ancho de banda
CREATE TABLE Mesura_bw (
  id_mesura       INT      NOT NULL AUTO_INCREMENT,
  id_operari      INT      NOT NULL,
  data_hora       DATETIME NOT NULL DEFAULT NOW(),
  download_mbps   FLOAT    NOT NULL,
  upload_mbps     FLOAT    NOT NULL,
  latencia_ms     FLOAT    NOT NULL,
  resultat        ENUM('acceptable','no acceptable') NOT NULL,
  notes           VARCHAR(300),
  PRIMARY KEY (id_mesura),
  CONSTRAINT fk_mesura_operari
    FOREIGN KEY (id_operari) REFERENCES Usuari_comunicacio(id_usuari)
    ON UPDATE CASCADE ON DELETE RESTRICT
);

-- Tabla de auditoría de seguridad (escrita por triggers)
CREATE TABLE Avis_auditoria (
  id_avis         INT           NOT NULL AUTO_INCREMENT,
  usuari_db       VARCHAR(100)  NOT NULL,
  taula_afectada  VARCHAR(100)  NOT NULL,
  operacio        VARCHAR(10)   NOT NULL,
  data_hora       DATETIME      NOT NULL DEFAULT NOW(),
  detall          VARCHAR(500),
  PRIMARY KEY (id_avis)
);

-- Control de copias de seguridad automáticas
CREATE TABLE Control_backup (
  id_backup   INT           NOT NULL AUTO_INCREMENT,
  data_hora   DATETIME      NOT NULL DEFAULT NOW(),
  taules      VARCHAR(300)  NOT NULL,
  resultat    ENUM('OK','ERROR') NOT NULL,
  PRIMARY KEY (id_backup)
);
```

---

### ✅ Integridad de Datos

Toda la base de datos utiliza restricciones `CASCADE` y `RESTRICT` de forma estratégica:

- **`ON UPDATE CASCADE`** → Si se actualiza una clave primaria referenciada, el cambio se propaga automáticamente a todas las tablas dependientes.
- **`ON DELETE RESTRICT`** → No se permite eliminar un registro que tenga otros registros dependientes, evitando datos huérfanos.

Verificación de que todas las tablas se han creado correctamente:

```sql
SHOW TABLES;
```

Resultado esperado:
```
+------------------------+
| Tables_in_innovatetech |
+------------------------+
| Avis_auditoria         |
| Control_backup         |
| Departament            |
| Empleat                |
| Grup_qualitat          |
| Mesura_bw              |
| Trucada                |
| Usuari_comunicacio     |
| Video                  |
+------------------------+
9 rows in set (0.000 sec)
```

---

## ⚡ TRIGGERS

Un **trigger** es un bloque de código SQL que se ejecuta automáticamente cuando ocurre una acción específica sobre una tabla (INSERT, UPDATE o DELETE). Su función principal es:

- Automatizar tareas de seguridad
- Validar reglas de negocio
- Mantener un registro de auditoría sin intervención manual

Se implementaron **4 triggers** principales en la base de datos InnovateTech.

---

### Trigger 1 — Cuota de Minutos Mensuales

**Objetivo:** Limitar el tiempo total de conversación por usuario al mes, según los límites definidos en su grupo de calidad.

**Funcionamiento:** Antes de insertar una nueva llamada, suma los minutos acumulados del usuario en el mes actual y los compara con el límite de su grupo. Si el límite se ha superado, registra la infracción en la tabla de auditoría y bloquea la llamada.

```sql
DELIMITER //
CREATE TRIGGER tr_quota_minuts_mensuals
  BEFORE INSERT ON Trucada
  FOR EACH ROW
  BEGIN
    DECLARE v_minuts_usats INT DEFAULT 0;
    DECLARE v_max_minuts   INT DEFAULT 600;

    SELECT max_minuts_mes INTO v_max_minuts
    FROM Grup_qualitat
    WHERE id_grup = NEW.id_grup;

    SELECT COALESCE(SUM(durada_total), 0) INTO v_minuts_usats
    FROM Trucada
    WHERE id_originador = NEW.id_originador
      AND MONTH(inici) = MONTH(NOW())
      AND YEAR(inici)  = YEAR(NOW());

    IF v_minuts_usats >= v_max_minuts THEN
      INSERT INTO Avis_auditoria (usuari_db, taula_afectada, operacio, detall)
      VALUES (USER(), 'Trucada', 'INSERT',
              CONCAT('Quota mensual superada. Usuari id=', NEW.id_originador,
                     '. Minuts usats: ', v_minuts_usats, '/', v_max_minuts));
      SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Quota de minuts mensuals superada. Trucada no permesa.';
    END IF;
  END//
DELIMITER ;
```

**Prueba del Trigger 1:**

```sql
-- Reducimos el límite a 10 minutos para probar sin esperar
UPDATE Grup_qualitat SET max_minuts_mes = 10 WHERE id_grup = 1;

-- Intentamos registrar una llamada de 11 minutos → debe bloquearse
INSERT INTO Trucada (id_originador, id_destinatari, inici, fi, durada_total, id_grup)
VALUES (1, 2, NOW(), NOW(), 11, 1);
-- ERROR 1644 (45000): Quota de minuts mensuals superada. Trucada no permesa.
```

---

### Trigger 2 — Cuota Máxima de Llamadas Diarias

**Objetivo:** Limitar la cantidad de llamadas que un usuario puede realizar en un mismo día.

**Funcionamiento:** Antes de insertar una nueva llamada, cuenta las llamadas realizadas hoy por ese usuario y las compara con el máximo permitido para su grupo. Si se ha alcanzado el límite, registra el intento en auditoría y cancela la operación.

```sql
DELIMITER //
CREATE TRIGGER tr_quota_trucades_diaries
  BEFORE INSERT ON Trucada
  FOR EACH ROW
  BEGIN
    DECLARE v_trucades_avui  INT DEFAULT 0;
    DECLARE v_max_trucades   INT DEFAULT 20;

    SELECT max_trucades_dia INTO v_max_trucades
    FROM Grup_qualitat
    WHERE id_grup = NEW.id_grup;

    SELECT COUNT(*) INTO v_trucades_avui
    FROM Trucada
    WHERE id_originador = NEW.id_originador
      AND DATE(inici) = CURDATE();

    IF v_trucades_avui >= v_max_trucades THEN
      INSERT INTO Avis_auditoria (usuari_db, taula_afectada, operacio, detall)
      VALUES (USER(), 'Trucada', 'INSERT',
              CONCAT('Quota diaria superada. Usuari id=', NEW.id_originador,
                     '. Trucades avui: ', v_trucades_avui, '/', v_max_trucades));
      SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Quota de trucades diaries superada. Trucada no permesa.';
    END IF;
  END//
DELIMITER ;
```

**Prueba del Trigger 2:**

```sql
-- Reducimos el límite a 0 llamadas para probar
UPDATE Grup_qualitat SET max_trucades_dia = 0 WHERE id_grup = 1;

-- Intentamos registrar cualquier llamada → debe bloquearse
INSERT INTO Trucada (id_originador, id_destinatari, inici, fi, durada_total, id_grup)
VALUES (1, 2, NOW(), NOW(), 5, 1);
-- ERROR 1644 (45000): Quota de trucades diaries superada. Trucada no permesa.
```

---

### Trigger 3 — Bloqueo de Usuarios

**Objetivo:** Impedir que usuarios con estado `bloquejat` realicen o reciban llamadas.

**Funcionamiento:** Antes de insertar una nueva llamada, comprueba el estado tanto del originador como del destinatario. Si cualquiera de los dos está bloqueado, escribe un aviso detallado en auditoría y aborta la operación.

```sql
DELIMITER //
CREATE TRIGGER tr_bloqueig_usuari
  BEFORE INSERT ON Trucada
  FOR EACH ROW
  BEGIN
    DECLARE v_estat_orig VARCHAR(20);
    DECLARE v_estat_dest VARCHAR(20);

    SELECT estat INTO v_estat_orig
    FROM Usuari_comunicacio
    WHERE id_usuari = NEW.id_originador;

    SELECT estat INTO v_estat_dest
    FROM Usuari_comunicacio
    WHERE id_usuari = NEW.id_destinatari;

    IF v_estat_orig = 'bloquejat' THEN
      INSERT INTO Avis_auditoria (usuari_db, taula_afectada, operacio, detall)
      VALUES (USER(), 'Trucada', 'INSERT',
              CONCAT('Trucada bloquejada: originador id=', NEW.id_originador, ' esta bloquejat'));
      SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Usuari bloquejat. No es pot realitzar la trucada.';
    END IF;

    IF v_estat_dest = 'bloquejat' THEN
      INSERT INTO Avis_auditoria (usuari_db, taula_afectada, operacio, detall)
      VALUES (USER(), 'Trucada', 'INSERT',
              CONCAT('Trucada bloquejada: destinatari id=', NEW.id_destinatari, ' esta bloquejat'));
      SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Destinatari bloquejat. No es pot realitzar la trucada.';
    END IF;
  END//
DELIMITER ;
```

**Prueba del Trigger 3:**

```sql
-- El usuario id=6 ya está en estado 'bloquejat'
-- Intentamos que realice una llamada → debe bloquearse
INSERT INTO Trucada (id_originador, id_destinatari, inici, fi, durada_total, id_grup)
VALUES (6, 1, NOW(), NOW(), 5, 1);
-- ERROR 1644 (45000): Usuari bloquejat. No es pot realitzar la trucada.
```

---

### Trigger 4 — Auditoría de Empleados (UPDATE y DELETE)

**Objetivo:** Vigilar y proteger los datos de la tabla de empleados registrando cualquier modificación y prohibiendo las eliminaciones directas.

**Funcionamiento:**
- **UPDATE:** Permite el cambio pero registra obligatoriamente un aviso en auditoría con el usuario de BD y el DNI afectado.
- **DELETE:** Registra el intento de eliminación en auditoría y detiene la operación con un error, ya que borrar empleados está prohibido por esta vía.

```sql
DELIMITER //

-- Trigger de UPDATE sobre Empleat
CREATE TRIGGER tr_auditoria_empleat_update
  BEFORE UPDATE ON Empleat
  FOR EACH ROW
  BEGIN
    INSERT INTO Avis_auditoria (usuari_db, taula_afectada, operacio, detall)
    VALUES (USER(), 'Empleat', 'UPDATE',
            CONCAT('UPDATE sobre Empleat per usuari: ', USER(),
                   '. DNI afectat: ', OLD.dni));
  END//

-- Trigger de DELETE sobre Empleat
CREATE TRIGGER tr_auditoria_empleat_delete
  BEFORE DELETE ON Empleat
  FOR EACH ROW
  BEGIN
    INSERT INTO Avis_auditoria (usuari_db, taula_afectada, operacio, detall)
    VALUES (USER(), 'Empleat', 'DELETE',
            CONCAT('Intent de DELETE a Empleat per usuari: ', USER(),
                   '. DNI afectat: ', OLD.dni));
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'Operació DELETE no permesa sobre la taula Empleat.';
  END//

DELIMITER ;
```

**Pruebas del Trigger 4:**

```sql
-- Prueba UPDATE: debe ejecutarse pero quedar registrado en auditoría
UPDATE Empleat SET telefon = '999999999' WHERE dni = '23456789B';

-- Verificamos el registro en auditoría
SELECT * FROM Avis_auditoria;

-- Prueba DELETE: debe bloquearse y quedar registrado
DELETE FROM Empleat WHERE dni = '89012345H';
-- ERROR 1644 (45000): Operació DELETE no permesa sobre la taula Empleat.

-- Verificamos que el intento quedó registrado
SELECT * FROM Avis_auditoria ORDER BY id_avis DESC LIMIT 5;
```

---


## Resumen de Automatizaciones Implementadas

| Elemento | Tipo | Tabla afectada | Acción |
|----------|------|----------------|--------|
| `tr_quota_minuts_mensuals` | TRIGGER BEFORE INSERT | `Trucada` | Bloquea llamadas si se supera la cuota mensual de minutos |
| `tr_quota_trucades_diaries` | TRIGGER BEFORE INSERT | `Trucada` | Bloquea llamadas si se supera el límite diario |
| `tr_bloqueig_usuari` | TRIGGER BEFORE INSERT | `Trucada` | Impide llamadas con usuarios bloqueados |
| `tr_auditoria_empleat_update` | TRIGGER BEFORE UPDATE | `Empleat` | Registra cualquier modificación de empleados |
| `tr_auditoria_empleat_delete` | TRIGGER BEFORE DELETE | `Empleat` | Prohíbe y registra intentos de borrado de empleados |

---

>  **Nota:** Todos los triggers escriben en la tabla `Avis_auditoria`, que actúa como registro centralizado de seguridad. Esto permite auditar cualquier incidencia o intento de violación de las reglas de negocio sin necesidad de revisar los logs del sistema operativo.

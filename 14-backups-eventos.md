## 💾 BACKUP Automático

Se programó un **evento automático** en la base de datos para realizar copias de seguridad diarias de todas las tablas principales, exportando los datos en formato CSV y registrando cada ejecución en la tabla `Control_backup`.

### Activar el Event Scheduler de MariaDB

```sql
-- Verificar estado
SHOW VARIABLES LIKE 'event_scheduler';

-- Activar si está desactivado
SET GLOBAL event_scheduler = ON;
```

### Creación del Evento de Backup Diario

```sql
DELIMITER //
CREATE EVENT ev_backup_diari
  ON SCHEDULE EVERY 1 DAY
  STARTS CURRENT_TIMESTAMP
  DO
  BEGIN
    SELECT * INTO OUTFILE '/tmp/backup_empleat.csv'
      FIELDS TERMINATED BY ',' ENCLOSED BY '"'
      LINES TERMINATED BY '\n'
      FROM Empleat;

    SELECT * INTO OUTFILE '/tmp/backup_departament.csv'
      FIELDS TERMINATED BY ',' ENCLOSED BY '"'
      LINES TERMINATED BY '\n'
      FROM Departament;

    SELECT * INTO OUTFILE '/tmp/backup_trucada.csv'
      FIELDS TERMINATED BY ',' ENCLOSED BY '"'
      LINES TERMINATED BY '\n'
      FROM Trucada;

    SELECT * INTO OUTFILE '/tmp/backup_usuari.csv'
      FIELDS TERMINATED BY ',' ENCLOSED BY '"'
      LINES TERMINATED BY '\n'
      FROM Usuari_comunicacio;

    SELECT * INTO OUTFILE '/tmp/backup_video.csv'
      FIELDS TERMINATED BY ',' ENCLOSED BY '"'
      LINES TERMINATED BY '\n'
      FROM Video;

    SELECT * INTO OUTFILE '/tmp/backup_mesura_bw.csv'
      FIELDS TERMINATED BY ',' ENCLOSED BY '"'
      LINES TERMINATED BY '\n'
      FROM Mesura_bw;

    SELECT * INTO OUTFILE '/tmp/backup_grup_qualitat.csv'
      FIELDS TERMINATED BY ',' ENCLOSED BY '"'
      LINES TERMINATED BY '\n'
      FROM Grup_qualitat;

    INSERT INTO Control_backup (data_hora, taules, resultat)
    VALUES (NOW(), 'Empleat, Departament, Trucada, Usuari_comunicacio, Video, Mesura_bw, Grup_qualitat', 'OK');
  END//
DELIMITER ;
```

### Permisos necesarios para escritura de ficheros

Para que el usuario root pueda escribir ficheros al disco, es necesario otorgarle el privilegio `FILE`:

```sql
GRANT FILE ON *.* TO 'root'@'localhost';
FLUSH PRIVILEGES;

-- Verificar que el permiso se ha asignado correctamente
SHOW GRANTS FOR 'root'@'localhost';
```

### Forzar un Backup Manual (sin esperar al evento)

```sql
-- Ejecutar manualmente cada exportación
SELECT * INTO OUTFILE '/tmp/backup_empleat.csv'
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\n'
  FROM Empleat;

SELECT * INTO OUTFILE '/tmp/backup_departament.csv'
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\n'
  FROM Departament;

SELECT * INTO OUTFILE '/tmp/backup_trucada.csv'
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\n'
  FROM Trucada;

SELECT * INTO OUTFILE '/tmp/backup_usuari.csv'
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\n'
  FROM Usuari_comunicacio;

SELECT * INTO OUTFILE '/tmp/backup_video.csv'
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\n'
  FROM Video;

SELECT * INTO OUTFILE '/tmp/backup_mesura_bw.csv'
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\n'
  FROM Mesura_bw;

SELECT * INTO OUTFILE '/tmp/backup_grup_qualitat.csv'
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\n'
  FROM Grup_qualitat;

-- Registrar manualmente el backup exitoso
INSERT INTO Control_backup (data_hora, taules, resultat)
VALUES (NOW(), 'Empleat, Departament, Trucada, Usuari_comunicacio, Video, Mesura_bw, Grup_qualitat', 'OK');
```

### Verificación del Backup

Desde la terminal del servidor, verificamos que los ficheros CSV se han generado correctamente:

```bash
# Listar los archivos generados
ls -lh /tmp/backup_*.csv
```

Resultado esperado:
```
-rw-r--r-- 1 mysql mysql  120 May 20 08:45 /tmp/backup_departament.csv
-rw-r--r-- 1 mysql mysql  681 May 20 08:45 /tmp/backup_empleat.csv
-rw-r--r-- 1 mysql mysql   99 May 20 08:45 /tmp/backup_grup_qualitat.csv
-rw-r--r-- 1 mysql mysql  433 May 20 08:45 /tmp/backup_mesura_bw.csv
-rw-r--r-- 1 mysql mysql  705 May 20 08:45 /tmp/backup_trucada.csv
-rw-r--r-- 1 mysql mysql  536 May 20 08:45 /tmp/backup_usuari.csv
-rw-r--r-- 1 mysql mysql  741 May 20 08:45 /tmp/backup_video.csv
```

```bash
# Visualizar el contenido de un archivo para confirmar que los datos son correctos
cat /tmp/backup_departament.csv
cat /tmp/backup_empleat.csv
```

Desde MariaDB, consultamos el historial de backups registrado:

```sql
SELECT * FROM Control_backup;
```

Resultado esperado:
```
+------------+---------------------+--------------------------------------------------------------------+---------+
| id_backup  | data_hora           | taules                                                             | resultat|
+------------+---------------------+--------------------------------------------------------------------+---------+
|          1 | 2025-05-15 02:00:00 | Empleat, Departament, Trucada                                      | OK      |
|          2 | 2026-05-20 08:32:25 | Empleat, Trucada, Usuari_comunicacio, Departament                  | OK      |
|          3 | 2026-05-20 08:42:59 | Empleat, Departament, Trucada, Usuari_comunicacio, Video, Mesura_bw| OK      |
+------------+---------------------+--------------------------------------------------------------------+---------+
```

---

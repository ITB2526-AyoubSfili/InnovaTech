# 3.2.6. Gestión de Datos y Seguridad de Usuarios

Este apartado detalla la fase final de la implementación: la carga de datos para pruebas de estrés, la automatización del alta de usuarios y la configuración del modelo de seguridad basado en roles.

---

## 1. Datos de Prueba (Dataset de Validación)

Se ha insertado un dataset completo para validar la integridad referencial y el correcto funcionamiento de los triggers de cuotas.

### SQL

```sql
-- 1. Departamentos y Empleados
INSERT INTO Departament (nom, telefon) VALUES
('Vendes', '931234001'),
('Suport Tècnic', '931234002'),
('Administració', '931234003');

INSERT INTO Empleat (dni, nom, cognoms, adreca, telefon, codi_dep) VALUES
('12345678A', 'Joan', 'García Pérez', 'Carrer Major 1, BCN', '612345001', 1),
('23456789B', 'Maria', 'Martínez López', 'Av. Diagonal 100, BCN', '612345002', 1);

-- 2. Configuración de Calidad (QoS)
INSERT INTO Grup_qualitat (nom, qualitat, max_minuts_mes, max_trucades_dia) VALUES
('Premium', 'alta', 1000, 50),
('Estàndard', 'mitja', 600, 20);

-- 3. Registro de Videollamadas (Pruebas de Triggers)
INSERT INTO Trucada (
    id_originador,
    id_destinatari,
    inici,
    fi,
    durada_total,
    id_grup
) VALUES (
    1,
    2,
    '2025-05-20 10:00:00',
    '2025-05-20 10:10:00',
    600,
    1
);
```

---

## 2. Script Avanzado de Creación de Usuarios (`crear_usuarios.sh`)

Con el objetivo de automatizar completamente la gestión de usuarios y minimizar errores administrativos, se ha desarrollado un script avanzado en Bash que genera dinámicamente los permisos correspondientes para cada rol del sistema y permite su ejecución inmediata sobre MariaDB.

### Características Principales

* Validación automática de parámetros de entrada.
* Comprobación de roles válidos.
* Generación automática de scripts SQL personalizados.
* Asignación de privilegios según el modelo RBAC definido.
* Posibilidad de ejecución inmediata sobre MariaDB.
* Registro temporal mediante nombres de archivo con timestamp.
* Salida coloreada para facilitar la supervisión de la ejecución.

### Código Fuente

```bash
#!/bin/bash

# ============================================================
# Script de creació automatitzada d'usuaris - InnovateTech
# Projecte Transversal ASIXc1 - Módul 0377
# ============================================================

# Colores para la salida
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Función de ayuda
mostrar_ayuda() {
    echo "Uso: $0 <usuario> <contraseña> <rol>"
    echo ""
    echo "Roles válidos:"
    echo "  - admin"
    echo "  - vendes"
    echo "  - administracio"
    echo "  - treballador"
    echo ""
    echo "Ejemplo:"
    echo "  $0 usuari1 Pass123! vendes"
    exit 1
}

# Verificar número de argumentos
if [ $# -ne 3 ]; then
    echo -e "${RED}Error: Número de argumentos incorrecto${NC}"
    mostrar_ayuda
fi

USUARIO=$1
PASSWORD=$2
ROL=$3

# Validar que el rol sea correcto
case $ROL in
    admin|vendes|administracio|treballador)
        ;;
    *)
        echo -e "${RED}Error: Rol '$ROL' no válido${NC}"
        mostrar_ayuda
        ;;
esac

# Generar timestamp para el nombre del archivo
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
ARCHIVO_SQL="crear_usuario_${USUARIO}_${TIMESTAMP}.sql"

echo -e "${YELLOW}Generando script SQL para el usuario: $USUARIO${NC}"
echo -e "${YELLOW}Rol asignado: $ROL${NC}"

# ============================================================
# Generar el archivo SQL
# ============================================================
cat > "$ARCHIVO_SQL" << EOF
-- ============================================================
-- Script generado automáticamente
-- Usuario: $USUARIO
-- Rol: $ROL
-- Fecha: $(date '+%Y-%m-%d %H:%M:%S')
-- ============================================================

-- Verificar si el usuario ya existe
SELECT User FROM mysql.user WHERE User = '$USUARIO';

-- Crear usuario
CREATE USER IF NOT EXISTS '$USUARIO'@'localhost' IDENTIFIED BY '$PASSWORD';
CREATE USER IF NOT EXISTS '$USUARIO'@'%' IDENTIFIED BY '$PASSWORD';

-- Asignar permisos según el rol
EOF

case $ROL in
    admin)
        cat >> "$ARCHIVO_SQL" << EOF

-- ROL: ADMIN - Acceso total
GRANT ALL PRIVILEGES ON innovatetech.* TO '$USUARIO'@'localhost';
GRANT ALL PRIVILEGES ON innovatetech.* TO '$USUARIO'@'%';

GRANT FILE ON *.* TO '$USUARIO'@'localhost';
GRANT FILE ON *.* TO '$USUARIO'@'%';
EOF
        ;;

    vendes)
        cat >> "$ARCHIVO_SQL" << EOF

-- ROL: VENDES - Puede gestionar clientes y llamadas
GRANT SELECT, INSERT, UPDATE ON innovatetech.Empleat TO '$USUARIO'@'localhost';
GRANT SELECT, INSERT, UPDATE ON innovatetech.Empleat TO '$USUARIO'@'%';

GRANT SELECT, INSERT, UPDATE ON innovatetech.Usuari_comunicacio TO '$USUARIO'@'localhost';
GRANT SELECT, INSERT, UPDATE ON innovatetech.Usuari_comunicacio TO '$USUARIO'@'%';

GRANT SELECT, INSERT, UPDATE ON innovatetech.Trucada TO '$USUARIO'@'localhost';
GRANT SELECT, INSERT, UPDATE ON innovatetech.Trucada TO '$USUARIO'@'%';

GRANT SELECT ON innovatetech.Grup_qualitat TO '$USUARIO'@'localhost';
GRANT SELECT ON innovatetech.Grup_qualitat TO '$USUARIO'@'%';

GRANT SELECT ON innovatetech.Video TO '$USUARIO'@'localhost';
GRANT SELECT ON innovatetech.Video TO '$USUARIO'@'%';
EOF
        ;;

    administracio)
        cat >> "$ARCHIVO_SQL" << EOF

-- ROL: ADMINISTRACIO - Gestiona empleados y departamentos
GRANT SELECT, INSERT, UPDATE ON innovatetech.Empleat TO '$USUARIO'@'localhost';
GRANT SELECT, INSERT, UPDATE ON innovatetech.Empleat TO '$USUARIO'@'%';

GRANT SELECT, INSERT, UPDATE ON innovatetech.Departament TO '$USUARIO'@'localhost';
GRANT SELECT, INSERT, UPDATE ON innovatetech.Departament TO '$USUARIO'@'%';

GRANT SELECT, INSERT, UPDATE ON innovatetech.Grup_qualitat TO '$USUARIO'@'localhost';
GRANT SELECT, INSERT, UPDATE ON innovatetech.Grup_qualitat TO '$USUARIO'@'%';

-- NO tiene acceso al sistema de llamadas de clientes
EOF
        ;;

    treballador)
        cat >> "$ARCHIVO_SQL" << EOF

-- ROL: TREBALLADOR - Solo lectura y registro de su propia actividad
GRANT SELECT ON innovatetech.Video TO '$USUARIO'@'localhost';
GRANT SELECT ON innovatetech.Video TO '$USUARIO'@'%';

GRANT SELECT ON innovatetech.Grup_qualitat TO '$USUARIO'@'localhost';
GRANT SELECT ON innovatetech.Grup_qualitat TO '$USUARIO'@'%';

GRANT SELECT, INSERT ON innovatetech.Trucada TO '$USUARIO'@'localhost';
GRANT SELECT, INSERT ON innovatetech.Trucada TO '$USUARIO'@'%';

-- NO puede modificar datos de empleados ni ver nóminas
EOF
        ;;
esac

cat >> "$ARCHIVO_SQL" << EOF

-- Aplicar cambios
FLUSH PRIVILEGES;

-- Verificar usuario creado
SELECT User, Host FROM mysql.user WHERE User = '$USUARIO';

-- ============================================================
-- FIN DEL SCRIPT
-- ============================================================
EOF

echo -e "${GREEN}✓ Script SQL generado: $ARCHIVO_SQL${NC}"

read -p "¿Ejecutar el script en MariaDB ahora? (s/n): " EJECUTAR

if [ "$EJECUTAR" = "s" ] || [ "$EJECUTAR" = "S" ]; then
    echo -e "${YELLOW}Ejecutando script en MariaDB...${NC}"
    sudo mysql -u root -p innovatetech < "$ARCHIVO_SQL"

    if [ $? -eq 0 ]; then
        echo -e "${GREEN}✓ Usuario '$USUARIO' creado correctamente${NC}"
    else
        echo -e "${RED}✗ Error al ejecutar el script${NC}"
        exit 1
    fi
else
    echo -e "${YELLOW}Script guardado. Puedes ejecutarlo manualmente con:${NC}"
    echo "  sudo mysql -u root -p innovatetech < $ARCHIVO_SQL"
fi

echo ""
echo -e "${GREEN}========== RESUMEN ==========${NC}"
echo "Usuario: $USUARIO"
echo "Rol: $ROL"
echo "Archivo SQL: $ARCHIVO_SQL"
echo -e "${GREEN}=============================${NC}"
```

### Ejemplo de Uso

```bash
chmod +x crear_usuarios.sh

./crear_usuarios.sh joan_vendes Pass123! vendes
```

### Resultado Esperado

```text
Generando script SQL para el usuario: joan_vendes
Rol asignado: vendes

✓ Script SQL generado: crear_usuario_joan_vendes_20250526_210015.sql

¿Ejecutar el script en MariaDB ahora? (s/n): s

✓ Usuario 'joan_vendes' creado correctamente
```

---

## 3. Modelo de Roles (RBAC)

Se ha implementado un control de acceso basado en roles para cumplir con la normativa de protección de datos (RGPD).

| Rol               | Descripción   | Permisos Principales                                                         |
| ----------------- | ------------- | ---------------------------------------------------------------------------- |
| **Admin**         | Superusuario  | Control total de la base de datos, backups, auditorías y gestión de usuarios |
| **Vendes**        | Comercial     | Gestión de clientes, llamadas y consulta de calidad de servicio              |
| **Administració** | RRHH          | Gestión de empleados, departamentos y grupos de calidad                      |
| **Treballador**   | Usuario final | Consulta de vídeos, calidad de servicio y registro de llamadas               |

### Comprobación de Privilegios

Para verificar los permisos asignados a un usuario:

```sql
SHOW GRANTS FOR 'joan_vendes'@'localhost';
```

---

## ✅ 4. Verificación de Funcionamiento

Se han realizado pruebas cruzadas para asegurar el correcto funcionamiento del sistema de seguridad.

### 🔒 Intento de Acceso No Autorizado

**Caso:**

Un usuario con rol **Vendes** intentó eliminar un registro de la tabla `Empleat`.

**Resultado:**

```text
ERROR 1142 (42000): DELETE command denied
```

El sistema bloqueó correctamente la operación al no disponer el usuario de permisos de eliminación.

### ⚡ Validación de Trigger de Cuotas

**Caso:**

Se intentó registrar una llamada que superaba el límite diario establecido para el grupo de calidad asignado.

**Resultado:**

* Inserción rechazada automáticamente.
* Activación del trigger de control.
* Registro del incidente en la tabla `Avis_auditoria`.
* Conservación de la integridad de los datos.

---

## Conclusiones

La implementación desarrollada proporciona:

* Integridad referencial mediante claves primarias y foráneas.
* Automatización segura de la gestión de usuarios.
* Aplicación estricta del modelo RBAC (Role Based Access Control).
* Protección frente a accesos no autorizados.
* Trazabilidad de operaciones mediante auditoría.
* Control automático de cuotas de uso mediante triggers.
* Compatibilidad con buenas prácticas de seguridad y cumplimiento RGPD.

La combinación de automatización, control de privilegios y mecanismos de auditoría garantiza un entorno robusto, escalable y seguro para la plataforma de comunicaciones de InnovateTech.

# Proyecto Final: Migración de Base de Datos (MariaDB a PostgreSQL)

* **Estudiante:** Carlos Daniel Jaure Lijeron
* **Asignatura:** Tecnología de Base de Datos I
* **Docente:** Ing. Jared Lopez
* **Fecha:** 28 de septiembre de 2026

---

## Punto 1: Migración de Tablas (Datos)

### 1.1 Exportar datos desde MariaDB
Para realizar la extracción de datos puros sin afectar la estructura ya migrada a PostgreSQL, se ejecutó un volcado de solo datos (`--no-create-info`) utilizando la herramienta `mariadb-dump` con credenciales de administrador. Se empleó el parámetro `--complete-insert` para declarar los nombres de las columnas explícitamente y garantizar la compatibilidad en el motor destino.

**Comando ejecutado en el entorno de origen:**
```bash
docker exec mariadb mariadb-dump -u root -p1727 --skip-ssl --no-create-info --complete-insert --compact employees > datos_mariadb.sql
```

### 1.2 Importar datos en PostgreSQL
Durante el primer intento de importación, PostgreSQL rechazó la sintaxis debido al uso de comillas invertidas (`) propias de MariaDB en las sentencias INSERT. Para solucionarlo y garantizar la compatibilidad, se utilizó la herramienta sed en Linux para limpiar el archivo antes de la inyección final en la base de datos pdb_employees.

**Comandos ejecutados (Limpieza e Importación):**

```Bash
sed -i 's/`//g' datos_mariadb.sql
docker exec -i postgresql psql -U carlos -d pdb_employees < datos_mariadb.sql
```

### 1.3 Verificación de carga (Conteo de filas)
Para garantizar la consistencia de los datos tras la corrección de integridad referencial, se ejecutó una consulta de consolidación (UNION ALL) en ambos motores de bases de datos para contabilizar el total de registros por tabla.

**Comando de validación y salida en MariaDB (Origen):**
```bash
docker exec -it mariadb mariadb -u root -p1727 --skip-ssl -D employees -e "SELECT 'departments' AS tabla, COUNT(*) FROM departments UNION ALL SELECT 'employees', COUNT(*) FROM employees UNION ALL SELECT 'dept_emp', COUNT(*) FROM dept_emp UNION ALL SELECT 'dept_manager', COUNT(*) FROM dept_manager UNION ALL SELECT 'salaries', COUNT(*) FROM salaries UNION ALL SELECT 'titles', COUNT(*) FROM titles;"
```
```text
+--------------+----------+
| tabla        | COUNT(*) |
+--------------+----------+
| departments  |        9 |
| employees    |   300024 |
| dept_emp     |   331603 |
| dept_manager |       24 |
| salaries     |  2844047 |
| titles       |   443308 |
+--------------+----------+
```

**Comando de validación y salida en PosgreSQL (Destino):**
```bash
docker exec -it postgresql psql -U carlos -d pdb_employees -c "SELECT 'departments' AS tabla, COUNT(*) FROM departments UNION ALL SELECT 'employees', COUNT(*) FROM employees UNION ALL SELECT 'dept_emp', COUNT(*) FROM dept_emp UNION ALL SELECT 'dept_manager', COUNT(*) FROM dept_manager UNION ALL SELECT 'salaries', COUNT(*) FROM salaries UNION ALL SELECT 'titles', COUNT(*) FROM titles;"
```

```text
tabla     |  count
--------------+---------
 departments  |       9
 employees    |  300024
 dept_emp     |  331603
 dept_manager |      24
 salaries     | 2844047
 titles       |  443308
(6 rows)
```

**Conclusión de la carga:** Como se observa en las salidas, el número de filas coincide exactamente en las 6 tablas entre el origen y el destino. La migración de datos ha sido 100% exitosa, demostrando que la solución de ordenar jerárquicamente la exportación resolvió el problema de integridad referencial detectado inicialmente.

## Punto 2: Migración de Vistas

### 2.1 Extracción de vistas en MariaDB
Para obtener la estructura original de las vistas, se consultaron los metadatos de MariaDB utilizando el comando SHOW CREATE VIEW.

**Comando ejecutado:**
```bash
docker exec -it mariadb mariadb -u root -p1727 --skip-ssl -D employees -e "SHOW CREATE VIEW current_dept_emp\G SHOW CREATE VIEW dept_emp_latest_date\G"
```

**Análisis de la salida:**
Se detectó que la vista current_dept_emp tiene una dependencia directa (un JOIN) con la vista dept_emp_latest_date. Por lo tanto, para la migración, es obligatorio establecer un orden de creación jerárquico. Además, el código original contenía parámetros exclusivos de MariaDB (ALGORITHM=UNDEFINED, DEFINER) y comillas invertidas (`).

### 2.2 Adaptación de la sintaxis a PostgreSQL
Para garantizar la compatibilidad con el motor de PostgreSQL 18, se realizó la siguiente adaptación del código DDL:

Se eliminaron las cláusulas propietarias de MariaDB (ALGORITHM, DEFINER, SQL SECURITY DEFINER).

Se removieron las comillas invertidas (`) para evitar errores de sintaxis en identificadores.

Se estandarizó la cláusula CREATE OR REPLACE VIEW.

### 2.3 Creación de las vistas en PostgreSQL
Aplicando el orden de dependencia correcto, se inyectó el código DDL adaptado directamente mediante el cliente psql.

**Comando ejecutado en PostgreSQL:**
```bash
docker exec -it postgresql psql -U carlos -d pdb_employees -c "
CREATE OR REPLACE VIEW dept_emp_latest_date AS
SELECT emp_no, MAX(from_date) AS from_date, MAX(to_date) AS to_date
FROM dept_emp
GROUP BY emp_no;

CREATE OR REPLACE VIEW current_dept_emp AS
SELECT l.emp_no, d.dept_no, l.from_date, l.to_date
FROM dept_emp d
JOIN dept_emp_latest_date l 
ON d.emp_no = l.emp_no AND d.from_date = l.from_date AND l.to_date = d.to_date;
"
```

**Salida de confirmación:**
```text
CREATE VIEW
CREATE VIEW
```

## Punto 3: Consultas de Verificación
### 3.1 y 3.2 Verificación de Integridad y Comparación

Para validar el éxito de la migración, se realizaron dos pruebas fundamentales: la verificación del funcionamiento de las vistas migradas (comparando resultados entre ambos motores) y una prueba de integridad referencial para descartar la existencia de registros huérfanos en las tablas dependientes.

**Prueba 1: Conteo de la vista current_dept_emp en MariaDB (Origen)**
```bash
docker exec -it mariadb mariadb -u root -p1727 --skip-ssl -D employees -e "SELECT COUNT(*) AS total_vista FROM current_dept_emp;"
```
```text
+-------------+
| total_vista |
+-------------+
|      300024 |
+-------------+
```

**Prueba 1: Conteo de la vista current_dept_emp en PostgreSQL (Destino)**
```bash
docker exec -it postgresql psql -U carlos -d pdb_employees -c "SELECT COUNT(*) AS total_vista FROM current_dept_emp;"
```
```text
total_vista 
-------------
      300024
(1 row)
```

**Prueba 2:** Búsqueda de registros huérfanos en PostgreSQL (Integridad referencial)
Se verificó que ningún registro en la tabla dept_emp estuviera asignado a un empleado inexistente en la tabla principal employees.
```bash
docker exec -it postgresql psql -U carlos -d pdb_employees -c "SELECT COUNT(*) AS registros_huerfanos FROM dept_emp WHERE emp_no NOT IN (SELECT emp_no FROM employees);"
```
```text
registros_huerfanos 
---------------------
                   0
(1 row)
```
### 3.3 Conclusiones de la Verificación
Tras ejecutar las pruebas de validación, se concluye que:

**Consistencia de Vistas:** La adaptación de la sintaxis DDL de las vistas a PostgreSQL fue exitosa. La vista current_dept_emp arroja exactamente 300,024 registros en ambos SGBD, demostrando que la lógica relacional original (los JOINs y agrupaciones) se preservó intacta.

**Integridad de los Datos:** La consulta de registros huérfanos retornó 0 filas, confirmando que la estrategia aplicada en el Punto 1.2 (forzar un orden jerárquico durante la exportación) garantizó al 100% la integridad referencial y evitó cualquier pérdida o corrupción de datos durante el traslado. La migración cumple exitosamente con todos los criterios técnicos requeridos.






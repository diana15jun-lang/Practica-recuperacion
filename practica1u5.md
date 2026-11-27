|<div align="center"><img src="../img/tec.png" width="130"/></div>|<div align="center"><img src="../img/itma.jpg" width="110"/></div>|
| :---: | :---: |
# Recuperación de un SGBD (MariaDB)

**Institución:** Tecnológico Nacional de Milpa Alta II  
**Materia:** Administración de Bases de Datos  
**Práctica:** Recuperación de un SGBD  
**Estudiante:** Diana Yareli Perez Martinez 
**Profesor:** Diego Alberto Bollas  
**Fecha:** 26 de noviembre de 2026  

## Introducción

La disponibilidad e integridad de los datos son pilares fundamentales en la administración de bases de datos. Un Sistema Gestor de Bases de Datos (SGBD).

En esta práctica se implementan dos estrategias complementarias de recuperación en MariaDB sobre Ubuntu (WSL):

1. **Respaldo y restauración lógica (`mariadb-dump`):** permite generar una copia portable de la estructura y los datos de una base de datos en un archivo `.sql` para reconstruirla tras una pérdida total.
2. **Recuperación a un punto en el tiempo (PITR) con Binary Log (`mariadb-binlog`):** permite recuperar transacciones posteriores al último respaldo lógico y detener la reproducción exactamente antes de un error humano.

- **Carpeta de trabajo:** `~/practica1`

```bash
mkdir practica1 && cd practica1
```

## Desarrollo

### Parte A: Respaldo y restauración con `mariadb-dump`

#### Paso 1. Creación de la base de datos y carga de datos iniciales

Se inició el servicio de MariaDB y se accedió al cliente con el usuario `root`:

```bash
sudo service mariadb start
mariadb -u root -p
```

Dentro del cliente se creó la base de datos `tienda`, la tabla `clientes`, se insertaron 5 registros iniciales y se verificó el conteo:

```sql
CREATE DATABASE tienda;
USE tienda;

CREATE TABLE clientes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO clientes (nombre, email) VALUES
('Ana Ramos', 'ana@correo.com'),
('Alex Lopez', 'alex@correo.com'),
('Mateo Martinez', 'mateo@correo.com'),
('Monse Martinez', 'monse@correo.com'),
('Daniela Martinez', 'dan@correo.com');

SELECT COUNT(*) FROM clientes;
EXIT;
```

**Resultado obtenido:** `Query OK, 5 rows affected` y `COUNT(*) = 5`.



#### Paso 2. Generación del respaldo lógico

Desde la terminal de Ubuntu se generó el respaldo completo de la base `tienda` con `mariadb-dump` y se verificó su tamaño:

```bash
mariadb-dump -u root -p --databases tienda > tienda_backup.sql
ls -lh tienda_backup.sql
```

**Resultado obtenido:** se creó el archivo `tienda_backup.sql` con un tamaño de `2.5K`.



#### Paso 3. Simulación del desastre (`DROP DATABASE`)

Se ingresó nuevamente a MariaDB para simular el borrado accidental de la base de datos y se listaron las bases existentes:

```bash
mariadb -u root -p
```

```sql
DROP DATABASE tienda;
SHOW DATABASES;
EXIT;
```

**Resultado obtenido:** la base de datos `tienda` desapareció del listado de bases del servidor.



#### Paso 4 y Paso 5. Restauración del respaldo y verificación

Se restauró el archivo `tienda_backup.sql` desde la terminal y se consultó el contenido de la tabla `tienda.clientes`:

```bash
mariadb -u root -p < tienda_backup.sql
mariadb -u root -p -e "SELECT * FROM tienda.clientes;"
```

**Resultado obtenido:** se recuperó la base de datos `tienda` y la tabla `clientes` mostró los 5 registros originales (`id` 1 al 5).



### Parte B: Recuperación a un punto en el tiempo (PITR) con Binary Log

#### Paso 1. Verificación del Binary Log (`log_bin`)

Se verificó que el registro binario estuviera habilitado en el servidor:

```bash
mariadb -u root -p -e "SHOW VARIABLES LIKE 'log_bin';"
mariadb -u root -p -e "SHOW VARIABLES LIKE 'log_bin_basename';"
```

**Resultado obtenido:**
- `log_bin = ON`
- `log_bin_basename = /var/lib/mariadb/mysql-bin`



#### Paso 2. Respaldo base con posición del binlog

Se generó un nuevo respaldo utilizando `--flush-logs` (para rotar el archivo binlog) y `--master-data=2` (para registrar la coordenada exacta del log en el archivo `.sql`), y se extrajo dicha coordenada con `grep`:

```bash
mariadb-dump -u root -p --databases tienda --flush-logs --master-data=2 > tienda_pitr.sql
grep "CHANGE MASTER" tienda_pitr.sql
```

**Resultado obtenido:**

```sql
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000016', MASTER_LOG_POS=385;
```

- **Archivo binlog:** `mysql-bin.000016`
- **Posición inicial:** `385`



#### Paso 3. Inserción de datos posteriores al respaldo

Se ingresó a MariaDB y se insertaron 2 clientes adicionales posteriores al respaldo:

```sql
USE tienda;
INSERT INTO clientes (nombre, email) VALUES
('Juan Romero', 'juan@correo.com'),
('Diana Perez', 'dian@correo.com');

SELECT COUNT(*) FROM clientes;
```

**Resultado obtenido:** `Query OK, 2 rows affected` y `COUNT(*) = 7`.

Tras esperar unos segundos, se consultó la hora exacta del servidor para usarla como punto de corte:

```sql
SELECT NOW();
```

**Resultado obtenido:** `2026-11-26 19:50:37`.



#### Paso 4. Simulación de borrado accidental (`DELETE`)

Después de esperar más de 5 segundos tras el `SELECT NOW()`, se simuló un borrado accidental sin `WHERE`:

```sql
DELETE FROM clientes;
SELECT COUNT(*) FROM clientes;
EXIT;
```

**Resultado obtenido:** `Query OK, 7 rows affected` y `COUNT(*) = 0`.



#### Paso 5. Restauración del respaldo base

Se restauró primero el respaldo lógico `tienda_pitr.sql` (que contiene únicamente los 5 clientes iniciales):

```bash
mariadb -u root -p < tienda_pitr.sql
```

**Resultado obtenido:** la tabla `clientes` volvió a tener 5 registros (aún faltaban Juan Romero y Diana Perez).

#### Paso 6 y Paso 7. Aplicación del binlog hasta antes del error y verificación final

Utilizando la posición `385`, la ruta real del binlog (`/var/lib/mariadb/mysql-bin.000016`) y la marca de tiempo `2026-11-26 19:50:37` (anterior al `DELETE`), se reprodujeron los eventos del binlog sobre MariaDB y se verificó la tabla:

```bash
sudo mariadb-binlog --start-position=385 --stop-datetime="2026-11-26 19:50:37" /var/lib/mariadb/mysql-bin.000016 | mariadb -u root -p1234
mariadb -u root -p -e "SELECT * FROM tienda.clientes;"
```

**Resultado obtenido:** se recuperaron los **7 clientes completos** (`Ana Ramos`, `Alex Lopez`, `Mateo Martinez`, `Monse Martinez`, `Daniela Martinez`, `Juan Romero` y `Diana Perez`), excluyendo el `DELETE` accidental.



## Conclusiones

1. Un respaldo lógico periódico con `mariadb-dump` es sencillo y portable, pero por sí solo implica perder cualquier transacción ocurrida entre el momento del respaldo y el incidente.
2. Activar el **Binary Log (`log_bin`)** y combinar respaldos con `--flush-logs --master-data=2` habilita la **recuperación a un punto en el tiempo (PITR)**, permitiendo restaurar tanto los datos base como los cambios posteriores hasta el segundo exacto previo a un error humano.
3. Verificar la ruta real de los archivos de registro (`log_bin_basename`) y mantener márgenes de tiempo claros durante la recuperación evita errores operativos y garantiza la consistencia de la base de datos.

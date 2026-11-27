# Cuestionario: Recuperación de una base de datos en MariaDB


## 1. ¿Qué datos se habrían perdido si solo hubieras usado el respaldo de la Parte A?

Se habrían perdido todos los datos registrados **después** de haber generado el archivo `tienda_backup.sql`. En esta práctica específica, al restaurar únicamente el respaldo base solo regresaban los 5 clientes iniciales (`Ana Ramos`, `Alex Lopez`, `Mateo Martinez`, `Monse Martinez` y `Daniela Martinez`), por lo que se habrían perdido los registros posteriores: **Juan Romero** (`juan@correo.com`) y **Diana Perez** (`dian@correo.com`), además de cualquier otra inserción, actualización o eliminación ocurrida entre el respaldo y el fallo.

## 2. ¿Para qué sirve la opción `--master-data=2`?

Sirve para que `mariadb-dump` incluya dentro del archivo `.sql` generado una línea comentada con la coordenada exacta del Binary Log en el momento en que se tomó el respaldo, por ejemplo:

```sql
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000016', MASTER_LOG_POS=385;
```

Al estar comentada (`--`), no afecta la restauración normal del respaldo, pero proporciona el archivo (`MASTER_LOG_FILE`) y la posición exacta (`MASTER_LOG_POS`) desde donde se debe comenzar a aplicar el binlog en una recuperación a un punto en el tiempo (PITR) o al inicializar una réplica, evitando omitir transacciones o aplicar eventos duplicados.

## 3. ¿Por qué es necesario activar el binlog **antes** de que ocurra el error?

Porque el Binary Log únicamente registra las operaciones de modificación (`INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, etc.) que ocurren **mientras está activado**. Si `log_bin` se encontraba apagado antes del incidente, MariaDB no habrá guardado ningún historial de las transacciones realizadas después del último respaldo, haciendo imposible reconstruir los cambios posteriores mediante PITR.

## 4. ¿Por qué se esperaron 5 segundos antes y después de anotar la hora?

Se esperaron 5 segundos antes y después para garantizar una **separación temporal clara** (en segundos distintos) entre las operaciones válidas (`INSERT` de los nuevos clientes), la marca de tiempo de referencia (`SELECT NOW()`) y la operación errónea (`DELETE FROM clientes`). Como `--stop-datetime` trabaja con granularidad de segundos, si el `INSERT` y el `DELETE` ocurrieran dentro del mismo segundo, detener el binlog por fecha y hora podría incluir el `DELETE` accidental o excluir los registros válidos.

# Anexo: Investigación de términos y herramientas de recuperación en MariaDB

**Materia:** Administración de Bases de Datos  
**Práctica:** Recuperación de un SGBD  

## 1. Conceptos fundamentales

### SGBD (Sistema Gestor de Bases de Datos)
Software encargado de crear, almacenar, consultar, proteger y administrar bases de datos garantizando la integridad, seguridad y concurrencia de la información. En esta práctica se utilizó **MariaDB**, un SGBD relacional de código abierto derivado de MySQL.

### InnoDB
Es el motor de almacenamiento transaccional por defecto en MariaDB. Soporta propiedades **ACID** (Atomicidad, Consistencia, Aislamiento y Durabilidad), bloqueo a nivel de fila, llaves foráneas y recuperación automática ante caídas (*crash recovery*).

### Respaldo lógico vs. Respaldo físico
- **Respaldo lógico:** exporta la estructura y los datos como sentencias SQL (`CREATE`, `INSERT`). Es legible y portable entre versiones, ideal para bases pequeñas o medianas, aunque más lento en volúmenes grandes.
- **Respaldo físico:** copia directamente los archivos binarios y directorios de datos del servidor en disco. Es mucho más rápido para bases de gran tamaño y soporta respaldos incrementales.

### PITR (Point-in-Time Recovery / Recuperación a un punto en el tiempo)
Técnica que combina un respaldo completo (lógico o físico) con la reproducción secuencial de los registros binarios (**binlog**) hasta una fecha/hora o posición exacta, permitiendo revertir errores humanos como un `DROP TABLE` o un `DELETE` sin `WHERE`.

### Mecanismos internos de recuperación ante caídas (*Crash Recovery*)
- **Redo log:** registra los cambios antes de aplicarse en los archivos de datos (*Write-Ahead Logging*). Permite rehacer (*REDO*) transacciones confirmadas (`COMMIT`) que aún no habían bajado a disco tras una caída.
- **Undo log:** guarda los valores previos a una modificación para deshacer (*UNDO*) transacciones incompletas o que ejecutaron `ROLLBACK`.
- **Checkpoint:** punto de control que indica hasta dónde los cambios en memoria ya fueron escritos de forma segura en disco.
- **Doublewrite buffer:** área intermedia que protege contra páginas corruptas o escritas a medias si el servidor se apaga durante una escritura.

### Binary Log (`binlog`)
Registro secuencial de todos los cambios realizados en la base de datos (`INSERT`, `UPDATE`, `DELETE`, `DDL`) en el orden en que fueron confirmados. Es la base tanto para **PITR** como para la **replicación** entre servidores.

### Métricas y reglas de estrategia de respaldo
- **RPO (Recovery Point Objective):** cantidad máxima de datos (medida en tiempo) que una organización puede tolerar perder tras un fallo.
- **RTO (Recovery Time Objective):** tiempo máximo tolerable que el sistema puede permanecer fuera de línea mientras se recupera.
- **Regla 3-2-1:** mantener al menos **3** copias de los datos, en **2** medios de almacenamiento distintos, con **1** copia almacenada fuera del sitio principal.

## 2. Herramientas y parámetros utilizados

| Herramienta / Opción | Función |
| :--- | :--- |
| `mariadb` | Cliente de línea de comandos para conectarse al servidor MariaDB y ejecutar sentencias SQL. |
| `mariadb-dump` | Utilidad para generar respaldos lógicos en formato `.sql` (en versiones anteriores a 10.5 se llamaba `mysqldump`). |
| `mariadb-binlog` | Herramienta para leer, filtrar y convertir archivos binarios del binlog a sentencias SQL aplicables (antes `mysqlbinlog`). |
| `mariadb-backup` | Herramienta oficial para respaldos físicos en caliente sin detener el servicio. |
| `--databases` | Indica a `mariadb-dump` que incluya las sentencias `CREATE DATABASE` y `USE` en el archivo de respaldo. |
| `--flush-logs` | Cierra el archivo de binlog actual y abre uno nuevo al iniciar el respaldo, facilitando la separación de eventos posteriores. |
| `--master-data=2` | Escribe como comentario en el respaldo el nombre del archivo binlog y la posición exacta (`MASTER_LOG_FILE` y `MASTER_LOG_POS`). |
| `--start-position` | Indica a `mariadb-binlog` el byte exacto desde donde comenzar a leer eventos dentro del archivo binlog. |
| `--stop-datetime` | Indica a `mariadb-binlog` la fecha y hora exacta en que debe detener la lectura para excluir eventos posteriores (como un `DELETE` accidental). |
| `log_bin_basename` | Variable de sistema de MariaDB que muestra la ruta base y prefijo donde el servidor almacena los archivos del binlog (por ejemplo, `/var/lib/mariadb/mysql-bin`). |
| `innodb_force_recovery` | Parámetro de último recurso (niveles 1 a 6) para iniciar MariaDB en modo de solo lectura cuando InnoDB presenta corrupción y poder rescatar datos con `mariadb-dump`. |

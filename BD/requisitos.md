
# REQUISITOS DE BASE DE DATOS PARA INGESTA BATCH (NIFI)

Esta configuración es la estándar para procesos de extracción tradicionales donde NiFi realiza consultas programadas mediante conectores JDBC.

| Requisito | Valor esperado | Observaciones |
|-----------|---------------|----------------|
| Host / IP | IP o Hostname accesible | Debe haber conectividad de red desde el nodo de NiFi. |
| Puerto | Puerto nativo del motor | Ej: 5432 (PG), 3306 (MySQL), 1433 (SQL Server). |
| Driver JDBC | Archivo .jar correspondiente | Debe cargarse en el servidor NiFi según el motor y versión. |
| Usuario | Usuario de Solo Lectura | Se requiere únicamente el privilegio SELECT. |
| Permisos | SELECT sobre tablas específicas | No se requieren permisos de superusuario ni de escritura. |
| Firewall | Puerto abierto hacia NiFi | Permitir tráfico desde la IP del [SERVIDOR]. |

---

# REQUISITOS DE BASE DE DATOS PARA INGESTA CDC (STREAMING KAFKA)

Esta configuración es obligatoria para la captura de cambios en tiempo real mediante Debezium y Kafka Connect, ya que requiere acceso a los registros lógicos del motor.

## POSTGRESQL

| Requisito | Valor esperado | Cómo verificarlo |
|-----------|---------------|------------------|
| Version | 10+ (Recomendado 12+) | `SELECT version();` |
| Host / IP | IP accesible desde el servidor | `ping <HOST>` |
| Puerto | 5432 (Default) | `nc -z <HOST> 5432` |
| Base de datos | Nombre de la BD origen | `\l` en psql |
| Usuario | Rol de Replicación y Login | Pedir al DBA |
| WAL Level | Estrictamente `logical` | `SHOW wal_level;` |
| Plugin | `pgoutput` | Nativo en versiones actuales |
| Replication Slots | `max_replication_slots` >= 1 | `SHOW max_replication_slots;` |
| Wal Senders | `max_wal_senders` >= 1 | `SHOW max_wal_senders;` |
| Privilegios | SELECT y REPLICATION | `ALTER USER <user> REPLICATION;` |

## MYSQL / MARIADB

| Requisito | Valor esperado | Cómo verificarlo |
|-----------|---------------|------------------|
| Version | 5.7+ (Recomendado 8.0+) | `SELECT VERSION();` |
| Host / IP | IP accesible desde el servidor | `ping <HOST>` |
| Puerto | 3306 (Default) | `nc -z <HOST> 3306` |
| Binary Log | `ON` | `SHOW VARIABLES LIKE 'log_bin';` |
| Binlog Format | Estrictamente `ROW` | `SHOW VARIABLES LIKE 'binlog_format';` |
| Binlog Row Image | Estrictamente `FULL` | `SHOW VARIABLES LIKE 'binlog_row_image';` |
| Retencion | Mínimo 3 días de logs | `SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';` |
| Privilegios | SELECT, REPLICATION SLAVE, REPLICATION CLIENT | `SHOW GRANTS FOR <user>;` |

## SQL SERVER

| Requisito | Valor esperado | Cómo verificarlo |
|-----------|---------------|------------------|
| Version | 2016+ (Standard/Enterprise) | `SELECT @@VERSION;` |
| Host / IP | IP accesible desde el servidor | `ping <HOST>` |
| Puerto | 1433 (Default) | `nc -z <HOST> 1433` |
| CDC en BD | `is_cdc_enabled = 1` | `SELECT is_cdc_enabled FROM sys.databases;` |
| CDC en Tablas | CDC activado por tabla | `EXEC sys.sp_cdc_help_change_data_capture;` |
| SQL Agent | Estado: RUNNING | Requerido para capturar cambios |
| Privilegios | db_owner o rol de captura | Pedir al DBA |

## MONGODB

| Requisito | Valor esperado | Cómo verificarlo |
|-----------|---------------|------------------|
| Version | 3.6+ (Recomendado 4.0+) | `db.version()` |
| Arquitectura | Replica Set (Obligatorio) | `rs.status()` |
| Oplog | Acceso de lectura a local.oplog.rs | Requerido para ver el flujo de cambios |
| Privilegios | Rol `read` en BD y en `local` | Pedir al DBA |

---

# RESUMEN 

```text

1. Motor de Base de Datos y Versión: (Ej. PostgreSQL 12, MySQL 8.0, SQL Server 2019)
2. Host / IP: (Accesible desde la red interna)
3. Puerto de Conexión:
4. Nombre de la Base de Datos:
5. Lista exacta de tablas / colecciones a sincronizar:
6. Credenciales de Acceso:
   - Usuario:
   - Contraseña:
7. Permisos de Red: Confirmar apertura del puerto del motor de BD hacia el servidor de integración (IP: 172.29.0.116).

Requisitos específicos según el método de extracción (A confirmar por el DBA):

[Para Ingesta Batch Tradicional - NiFi]
- El usuario proporcionado solo necesita permisos de lectura (SELECT). No requiere configuraciones a nivel de motor.

[Para Ingesta Streaming CDC - Kafka/Debezium]
- El usuario proporcionado requiere privilegios de REPLICACIÓN (acceso a lectura de logs transaccionales).
- PostgreSQL: El parámetro wal_level debe estar en 'logical'.
- MySQL: Binary log activado (log_bin=ON, binlog_format=ROW, binlog_row_image=FULL).
- SQL Server: CDC habilitado explícitamente a nivel de base de datos y a nivel de las tablas requeridas.
- MongoDB: El servidor debe estar configurado como Replica Set (no standalone) con acceso al oplog.
```

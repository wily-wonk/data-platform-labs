

# ipeline ETL Multifuente (NiFi - HDFS - Hive)

Este documento describe la implementación de un flujo de datos robusto para la extracción de información desde múltiples gestores de bases de datos relacionales, su ingesta en el Data Lake (HDFS) y su posterior disponibilidad para análisis en Apache Hive, resolviendo incompatibilidades críticas de entorno.



## 1. Arquitectura del Sistema
* **Orígenes de Datos:** PostgreSQL 10, MySQL 5.7 y SQL Server 2017 (Contenedores Docker en Host).
* **Orquestador:** Apache NiFi 1.28 (Servidor Debian).
* **Almacenamiento:** Hadoop HDFS (Formato Avro).
* **Procesamiento y Consulta:** Apache Hive 4.0.0 (Java 17).

## 2. Configuración de Conectividad y Seguridad

### 2.1. Reglas de Firewall (Windows Host)
Para permitir que el servidor Debian acceda a las bases de datos en el equipo de desarrollo, se habilitaron los puertos de entrada mediante PowerShell (Admin):

```powershell
New-NetFirewallRule -DisplayName "UADI_Ingesta_Postgres" -Direction Inbound -LocalPort 5433 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "UADI_Ingesta_MySQL" -Direction Inbound -LocalPort 3307 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "UADI_Ingesta_SQLServer" -Direction Inbound -LocalPort 1434 -Protocol TCP -Action Allow
```

### 2.2. Gestión de Controladores JDBC
Se instalaron los conectores necesarios en el servidor NiFi para asegurar la comunicación con los diversos motores:

```bash
# Descarga y ubicación de conectores
cd /tmp
wget https://jdbc.postgresql.org/download/postgresql-42.6.0.jar
wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.3.0/mysql-connector-j-8.3.0.jar
wget https://repo1.maven.org/maven2/com/microsoft/sqlserver/mssql-jdbc/12.6.1.jre11/mssql-jdbc-12.6.1.jre11.jar

sudo mv *.jar /opt/nifi/lib/
sudo chown nifi:nifi /opt/nifi/lib/*.jar
sudo chmod 644 /opt/nifi/lib/*.jar
```

## 3. Implementación del Flujo de Ingesta (NiFi)

### 3.1. Servicios de Controlador (DBCPConnectionPool)
Se configuraron pools de conexiones estandarizados:
* **PostgreSQL:** `jdbc:postgresql://<IP_HOST>:5433/negocios_db` | Driver: `org.postgresql.Driver`
* **MySQL:** `jdbc:mysql://<IP_HOST>:3307/negocios_db` | Driver: `com.mysql.cj.jdbc.Driver`
* **SQL Server:** `jdbc:sqlserver://<IP_HOST>:1434;databaseName=master;encrypt=true;trustServerCertificate=true;` | Driver: `com.microsoft.sqlserver.jdbc.SQLServerDriver`

### 3.2. Lógica de Extracción y Transformación
Se utilizaron procesadores `QueryDatabaseTable` para realizar cargas incrementales basadas en columnas de control (`id`, `cliente_id`, `fecha_creacion`). Posteriormente, se empleó `UpdateAttribute` para añadir metadatos de trazabilidad:
* `uadi.origen`: Identificador del gestor fuente.
* `uadi.tipo_dato`: Categoría de la información (Proveedores, Clientes, Productos).
* `uadi.fecha_ingesta`: Estampa de tiempo `${now()}`.

## 4. Almacenamiento en Data Lake (HDFS)

Los datos se depositan en el clúster de Hadoop bajo una estructura de directorios dinámica:
* **Ruta de destino:** `/ingesta/relacional/${uadi.tipo_dato}`
* **Formato:** Avro (Binario comprimido con esquema embebido).

Permisos aplicados en HDFS para permitir la escritura desde NiFi:
```bash
hdfs dfs -mkdir -p /ingesta/relacional
hdfs dfs -chmod -R 777 /ingesta
```

## 5. Integración con Apache Hive y Resolución de Problemas

Para habilitar el análisis de datos, se transfirieron los archivos al Warehouse de Hive:
```bash
hdfs dfs -mkdir -p /user/hive/warehouse/proveedores_test
hdfs dfs -cp /ingesta/relacional/Proveedores/* /user/hive/warehouse/proveedores_test/
```

### 5.1. Solución al Error de Serialización (Kryo & Java 17)
Se identificó un error `InaccessibleObjectException` debido al encapsulamiento de módulos en Java 17. Se aplicó una corrección en el entorno de Hive:

**Archivo:** `/usr/local/hive/conf/hive-env.sh`
```bash
JAVA17_OPENS="--add-opens java.base/java.util.concurrent.atomic=ALL-UNNAMED ..."
export HADOOP_OPTS="$HADOOP_OPTS $JAVA17_OPENS"
export HIVE_OPTS="" # Vacuna para evitar conflictos en systemd
```



### 5.2. Optimización del Motor de Ejecución
Ante fallos de memoria en YARN (`Return code 2`), se configuró el procesamiento en **Modo Local** para ejecutar consultas de agregación sin saturar el gestor de recursos:

**Configuración en Beeline:**
```sql
SET mapreduce.framework.name=local;

CREATE TABLE proveedores_test (
    id INT, nombre STRING, nit STRING, fecha_registro STRING
) STORED AS AVRO LOCATION '/user/hive/warehouse/proveedores_test';

SELECT COUNT(*) FROM proveedores_test;
```

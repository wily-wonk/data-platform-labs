# Integración de Capas de Almacenamiento y Análisis (Hive-HBase)

Este documento detalla la implementación de una capa de abstracción relacional sobre una base de datos NoSQL. El objetivo es permitir que los analistas de datos realicen consultas complejas mediante SQL (Hive) sobre datos ingeridos a alta velocidad en tiempo real (HBase), sin necesidad de duplicar el almacenamiento.

## 1. Abstracción SQL sobre NoSQL (Definición DDL)

Para habilitar la lectura de los datos de streaming en Hive, se crea una tabla externa que funciona como una "lupa" sobre el espacio de nombres de HBase.

```sql
CREATE EXTERNAL TABLE iot_data_streaming (
    row_key STRING,
    device_id STRING,
    humidity DOUBLE,
    pressure DOUBLE,
    processed_at STRING,
    status STRING,
    temperature DOUBLE,
    ts STRING,
    vibration DOUBLE
)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES (
    "hbase.columns.mapping" = ":key,data:device_id,data:humidity,data:pressure,data:processed_at,data:status,data:temperature,data:timestamp,data:vibration"
)
TBLPROPERTIES ("hbase.table.name" = "streamsim:iot_data");
```

## 2. Desglose Técnico de los Parámetros

| Componente | Función Técnica |
| :--- | :--- |
| **STORED BY** | Instruye a Hive para delegar la gestión del almacenamiento al manejador específico de HBase, evitando la búsqueda de archivos en HDFS. |
| **:key** | Mapea la clave primaria física (RowKey) de HBase a una columna lógica en Hive para permitir filtros indexados. |
| **data:column** | Define el mapeo entre la familia de columnas (`data`) y el cualificador de HBase hacia el esquema relacional de Hive. |
| **hbase.table.name** | Especifica el origen de datos exacto, incluyendo el Namespace (`streamsim`) y el identificador de la tabla. |

## 3. Consultas de Analítica en Tiempo Real

Una vez establecida la conexión, se pueden ejecutar operaciones de agregación complejas sobre el flujo de datos entrante.

### 3.1. Validación de Ingesta
Muestra los registros más recientes procesados por el pipeline de streaming.

```sql
SELECT * FROM iot_data_streaming LIMIT 5;
```

### 3.2. Agregación y Diagnóstico
Genera métricas de estado y promedios de temperatura para identificar anomalías en los dispositivos IoT.

```sql
SELECT 
    status, 
    COUNT(*) as total_eventos,
    AVG(temperature) as temp_promedio
FROM iot_data_streaming
GROUP BY status;
```

## 4. Configuración del Entorno (Troubleshooting)

Para asegurar la comunicación entre los servicios en el servidor único bajo Java 17, se deben inyectar las librerías de HBase en el entorno de ejecución de Hive.

**Paso 1: Edición del archivo de entorno**

| Comando de Configuración |
| :--- |
| `sudo nano /usr/local/hive/conf/hive-env.sh` |

**Paso 2: Inyección de Dependencias**
Añadir la siguiente variable al final del archivo para cargar los archivos `.jar` necesarios:

| Variable de Entorno |
| :--- |
| `export HIVE_AUX_JARS_PATH="/usr/local/hbase/lib/*"` |

**Paso 3: Reinicio del Servicio**
Es necesario reiniciar el demonio para que `systemd` cargue la nueva configuración de la JVM.

| Comando de Reinicio |
| :--- |
| `sudo systemctl restart hive-server2.service` |

---



# Troubleshooting: Errores de Ejecución y Serialización en Apache Hive (Java 17)

Este documento detalla la resolución de los problemas de compilación y ejecución de tareas distribuidas en Apache Hive 4.0.0 cuando se opera bajo el entorno estricto de Java 17. Se abordan los errores de encapsulamiento de módulos (Kryo) y las fallas de contenedores en YARN durante consultas de agregación.

## 1. Error de Serialización Kryo (Encapsulamiento Java 17)

**El Problema:**
Al ejecutar consultas que requieren el empaquetado de tareas (`COUNT`, `SUM`), Hive lanza un error crítico antes de enviar la instrucción al clúster:
`Error caching map.xml`
`Caused by: java.lang.reflect.InaccessibleObjectException: Unable to make field private volatile int java.util.concurrent.atomic.AtomicBoolean.value accessible...`

**La Causa:**
Java 17 introdujo un sistema estricto de encapsulamiento de módulos de seguridad. Apache Hive utiliza la librería "Kryo" para serializar consultas y enviarlas a los nodos. Al no tener permisos explícitos del sistema operativo para acceder a la librería `java.util.concurrent.atomic`, el empaquetado de la tarea de MapReduce se corrompe.

**La Solución:**
Inyectar banderas (flags) de máquina virtual `--add-opens` en la configuración del entorno para forzar la apertura de las librerías necesarias, manteniendo intacta la variable `HIVE_OPTS` para evitar colapsos en el servicio `systemd`.

Editar el archivo de entorno:
```bash
sudo nano /usr/local/hive/conf/hive-env.sh
```

Añadir/Modificar el bloque de configuración:
```bash
# Flags para abrir módulos encapsulados de Java 17 (Incluyendo librerías atómicas para Kryo)
JAVA17_OPENS="--add-opens java.base/java.lang=ALL-UNNAMED \
  --add-opens java.base/java.net=ALL-UNNAMED \
  --add-opens java.base/java.nio=ALL-UNNAMED \
  --add-opens java.base/java.util=ALL-UNNAMED \
  --add-opens java.base/java.util.concurrent=ALL-UNNAMED \
  --add-opens java.base/java.util.concurrent.atomic=ALL-UNNAMED \
  --add-opens java.base/sun.nio.ch=ALL-UNNAMED \
  --add-opens java.security.jgss/sun.security.krb5=ALL-UNNAMED"

export HADOOP_CLIENT_OPTS="$JAVA17_OPENS"
export HIVE_SERVER2_HADOOP_OPTS="$JAVA17_OPENS"
export HADOOP_OPTS="$HADOOP_OPTS $JAVA17_OPENS"

# Prevenir que el servicio systemd malinterprete los flags
export HIVE_OPTS=""
```

Reiniciar el servicio para aplicar los cambios:
```bash
sudo systemctl restart hive-server2
```

## 2. Falla de Tarea MapReduce (Síndrome del Contenedor Aislado)

**El Problema:**
Una vez resuelto el error de compilación (Kryo), la tarea es enviada a YARN, pero falla durante la ejecución con el siguiente registro:
`FAILED: Execution Error, return code 2 from org.apache.hadoop.hive.ql.exec.mr.MapRedTask`

**La Causa:**
En arquitecturas de un solo nodo (Single-Node Cluster) donde convergen múltiples servicios (NiFi, PostgreSQL, HDFS, YARN), el gestor de recursos carece de memoria RAM física para instanciar nuevos contenedores. Adicionalmente, los contenedores aislados de MapReduce que logra crear YARN nacen sin heredar la configuración `--add-opens` de Java 17, provocando un fallo interno al intentar deserializar datos binarios (Avro).

**La Solución:**
Configurar el motor de ejecución de Apache Hive en "Modo Local". Este enfoque elude al gestor YARN y obliga a que las operaciones de MapReduce se procesen directamente en la memoria JVM del propio HiveServer2 (el cual ya posee los permisos de Java 17 aplicados en el Paso 1).

**Solución por sesión (Para pruebas en Beeline):**
```sql
SET mapreduce.framework.name=local;
SELECT COUNT(*) FROM proveedores_test;
```

**Solución Permanente (Configuración a nivel clúster):**
Editar el archivo maestro de configuración de Hive:
```bash
sudo nano /usr/local/hive/conf/hive-site.xml
```

Añadir la propiedad en el bloque XML:
```xml
<property>
    <name>mapreduce.framework.name</name>
    <value>local</value>
</property>
```

Posterior a este cambio, reiniciar el servicio correspondiente (`sudo systemctl restart hive-server2.service`).

## 3. Verificación de Resultados
Con ambas soluciones aplicadas, las consultas de agregación sobre archivos binarios masivos procedentes de procesos ETL se completan exitosamente:

```sql
0: jdbc:hive2://localhost:10000> SELECT COUNT(*) FROM proveedores_test;
+-------+
|  _c0  |
+-------+
| 1000  |
+-------+
1 row selected (2.145 seconds)
```

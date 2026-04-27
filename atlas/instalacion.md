
# Guía Completa: Instalación y Configuración de Apache Atlas 2.4.0

Esta guía documenta el despliegue de **Apache Atlas 2.4.0** en un servidor único, configurado para integrarse con un ecosistema Apache externo (HBase, Solr, Kafka y Hive) y gestionando la coexistencia de múltiples versiones de Java (11 y 17).

---

## Diagnóstico del Ecosistema Base
Al inicio del despliegue, el servidor contaba con los siguientes servicios activos:

* **Java 17 (Global):** `/usr/lib/jvm/jdk-17.0.2/`
* **Hadoop HDFS/YARN:** 3.4.0
* **HBase:** 2.5.13 (Ruta: `/usr/local/hbase`)
* **Kafka:** 3.7.2 (Puerto: 9092)
* **Hive Server2:** 4.0.0
* **ZooKeeper:** 3.8.6 (Puerto: 2181)
* **NiFi:** 1.28.1

---

## Paso 1: Instalación y Coexistencia de Java 11

Apache Atlas 2.4.0 requiere estrictamente Java 11. Para no afectar al resto del ecosistema que corre sobre Java 17, se instala Java 11 y se aísla su ruta.

### 1.1 Instalación
```bash
sudo apt update
sudo apt install openjdk-11-jdk -y
```

### 1.2 Verificación de Rutas
Tras la instalación, el servidor mantiene dos entornos separados:
* **Java 11 (Solo para Atlas/Solr):** `/usr/lib/jvm/java-11-openjdk-amd64`
* **Java 17 (Global para Spark/Hive):** `/usr/lib/jvm/jdk-17.0.2/`

---

## Paso 2: Configuración de Apache Solr 8.11.3 (Externo)

Se utiliza Solr 8.11.3 (rama 8) por compatibilidad estricta con las APIs de JanusGraph integradas en Atlas.

### 2.1 Descarga y Preparación
```bash
cd /tmp
wget [https://archive.apache.org/dist/lucene/solr/8.11.3/solr-8.11.3.tgz](https://archive.apache.org/dist/lucene/solr/8.11.3/solr-8.11.3.tgz)
sudo tar -xvf solr-8.11.3.tgz -C /usr/local/
sudo mv /usr/local/solr-8.11.3 /usr/local/solr

# Crear usuario de servicio
sudo useradd -r -s /bin/false solr
sudo chown -R solr:solr /usr/local/solr
```

### 2.2 Configuración del Servicio Systemd
Se crea el archivo `/etc/systemd/system/solr.service` forzando el uso de la ruta específica de Java 11:
```ini
[Unit]
Description=Apache Solr en modo Cloud
After=network.target zookeeper.service

[Service]
Type=forking
User=solr
Group=solr
Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
ExecStart=/usr/local/solr/bin/solr start -c -z localhost:2181 -p 8983 -noprompt
ExecStop=/usr/local/solr/bin/solr stop -p 8983
LimitNOFILE=65000
LimitNPROC=65000
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 2.3 Creación de Colecciones Obligatorias
Levantar el servicio y crear los índices:
```bash
sudo systemctl daemon-reload
sudo systemctl enable solr
sudo systemctl start solr

# Ejecutar como usuario solr
sudo su - solr -s /bin/bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64

/usr/local/solr/bin/solr create -c vertex_index -shards 1 -replicationFactor 1
/usr/local/solr/bin/solr create -c edge_index -shards 1 -replicationFactor 1
/usr/local/solr/bin/solr create -c fulltext_index -shards 1 -replicationFactor 1
exit
```

---

## Paso 3: Compilación de Apache Atlas 2.4.0

Se realiza la compilación desde el código fuente excluyendo los componentes embebidos.

```bash
# Definir entorno de compilación temporal con Java 11
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export MAVEN_OPTS="-Xms2g -Xmx2g"

# Extraer fuentes y compilar
cd /tmp/apache-atlas-sources-2.4.0
mvn clean -DskipTests package -Pdist

# Despliegue del binario final
sudo tar -xzvf distro/target/apache-atlas-2.4.0-bin.tar.gz -C /usr/local/
sudo mv /usr/local/apache-atlas-2.4.0 /usr/local/atlas
sudo chown -R $USER:$USER /usr/local/atlas
```

---

## Paso 4: Configuración de Integración

Se modifican los archivos principales para aislar la JVM y conectar Atlas con los servicios on-premise.

### 4.1 Variables de Entorno (`atlas-env.sh`)
Archivo: `/usr/local/atlas/conf/atlas-env.sh`
```bash
# Asignación de Java 11 y desactivación de servicios locales
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export MANAGE_LOCAL_HBASE=false
export MANAGE_LOCAL_SOLR=false
export HBASE_CONF_DIR=/usr/local/hbase/conf
```

### 4.2 Propiedades de la Aplicación (`atlas-application.properties`)
Archivo: `/usr/local/atlas/conf/atlas-application.properties`
```properties
# HBase (Graph Storage)
atlas.graph.storage.backend=hbase
atlas.graph.storage.hostname=localhost

# Solr (Index Search)
atlas.graph.index.search.backend=solr
atlas.graph.index.search.solr.mode=cloud
atlas.graph.index.search.solr.zookeeper-url=localhost:2181

# Kafka (Notificaciones y Hooks)
atlas.notification.embedded=false
atlas.kafka.zookeeper.connect=localhost:2181
atlas.kafka.bootstrap.servers=localhost:9092
```

---

## Paso 5: Solución de Errores Críticos (Troubleshooting)

Durante el despliegue se resolvieron los siguientes bloqueos:

### 5.1 Conflicto de Librerías SLF4J (JVM Crash)
* **Error:** `ExceptionInInitializerError` causado por `Detected both log4j-over-slf4j.jar AND bound slf4j-log4j12.jar`.
* **Causa:** Atlas 2.4.0 usa Logback, pero las dependencias de cliente de HBase incluyen Reload4j/Log4j12, creando un bucle infinito en el logger.
* **Solución:** Renombrar/desactivar los archivos conflictivos en Atlas.
```bash
sudo find /usr/local/atlas -type f \( -name "slf4j-reload4j*.jar" -o -name "slf4j-log4j12*.jar" \) -exec mv {} {}.bak \;
```

### 5.2 Error de Resolución de Host en HBase
* **Error:** `Unable to resolve address: localhost2181:2181` en el log de aplicación.
* **Causa:** La propiedad `atlas.graph.storage.hostname` incluía el puerto (`localhost:2181`), provocando que Zookeeper concatenara el puerto dos veces.
* **Solución:** Limpiar la propiedad para que solo declare `localhost`.

### 5.3 Error 503 Service Unavailable (Conflicto Zookeeper)
* **Error:** El servidor web inicia, pero devuelve 503 por un fallo en el contexto de Spring (`Attempt 2: Starting zookeeper at localhost:2181 failed`).
* **Causa:** Atlas intentaba levantar su Zookeeper embebido en el puerto 2181, chocando con el Zookeeper principal del clúster.
* **Solución:** Forzar `atlas.notification.embedded=false` en el archivo de propiedades.

### 5.4 Firewall (UFW)
Se habilitó el acceso externo al puerto de la interfaz web:
```bash
sudo ufw allow 21000/tcp
```

---

## Paso 6: Arranque y Verificación

Para iniciar el servicio con logs limpios:
```bash
/usr/local/atlas/bin/atlas_start.py
```

Monitorizar el arranque:
```bash
tail -f /usr/local/atlas/logs/application.log
```
*Esperar el mensaje: `Apache Atlas Server started!!!`*

**Acceso a la Interfaz Web:**
* **URL:** `http://<IP_DEL_SERVIDOR>:21000`
* **Usuario:** `admin`
* **Contraseña:** `admin`
```

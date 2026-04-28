# Guía Técnica: Despliegue de Apache Atlas 2.4.0 en Ecosistema Híbrido Big Data


## 1. Introducción y Arquitectura de Aislamiento
Esta guía documenta la instalación de Apache Atlas 2.4.0 integrado con HBase 2, Solr 9 y Kafka. 

Debido a que el motor de grafos interno de Atlas (**JanusGraph**) utiliza métodos de reflexión sobre campos eliminados en Java 12 (`NoSuchFieldException: modifiers`), Atlas **no puede ejecutarse** en Java 17. 

**Solución:** El clúster opera globalmente con **Java 17** (Kafka, HBase), pero se implementa una "Burbuja de Ejecución" en **Java 11** exclusivamente para compilar Atlas, ejecutar el servicio y levantar Apache Solr.

---

## 2. Fase 1: Preparación del Entorno (Usuario: `uitga`)

### 2.1. Instalar Java 11 y Herramientas
```bash
sudo apt update
sudo apt install openjdk-11-jdk openjdk-17-jdk maven python3 -y
```

### 2.2. Despliegue de Solr 9.7.0 (Java 11)
```bash
cd /tmp
wget https://archive.apache.org/dist/solr/solr/9.7.0/solr-9.7.0.tgz
sudo tar -xzf solr-9.7.0.tgz -C /usr/local/
sudo mv /usr/local/solr-9.7.0 /usr/local/solr

# Usuario de servicio
sudo useradd -r -m -s /bin/bash -d /home/solr solr
sudo chown -R solr:solr /usr/local/solr
```

En `/etc/systemd/system/solr.service`, aislar la ejecución a Java 11:
```ini
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
ExecStart=/usr/local/solr/bin/solr start -c -z localhost:2181 -p 8983 -noprompt
```

### 2.3. Crear Colecciones en Solr
```bash
sudo su - solr -c "export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64; /usr/local/solr/bin/solr create -c vertex_index -shards 1 -replicationFactor 1"
sudo su - solr -c "export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64; /usr/local/solr/bin/solr create -c edge_index -shards 1 -replicationFactor 1"
sudo su - solr -c "export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64; /usr/local/solr/bin/solr create -c fulltext_index -shards 1 -replicationFactor 1"
```

---

## 3. Fase 2: Compilación de Atlas (Usuario: `hadoop_user`)

Descargar las fuentes de Atlas 2.4.0 en `/tmp/` y aplicar los siguientes parches para asegurar la compatibilidad con el entorno de compilación:

### 3.1. Parche del POM (`webapp/pom.xml`)
Se debe omitir el plugin `enunciate` agregando `<skip>true</skip>`:
```xml
<plugin>
    <groupId>com.webcohesion.enunciate</groupId>
    <artifactId>enunciate-maven-plugin</artifactId>
    <configuration>
        <skip>true</skip>
    </configuration>
</plugin>
```

### 3.2. Comando de Compilación (Maven Options)
Para evitar el error del plugin `sortpom` y permitir la compilación forzada:
```bash
export MAVEN_OPTS="-Xms2g -Xmx4g --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED"

mvn package -DskipTests -Pdist -Dmaven.javadoc.skip=true -Drat.skip=true -Dsort.skip=true
```

Al finalizar, instalar los binarios resultantes en `/usr/local/atlas` y asignar permisos a `hadoop_user`.

---

## 4. Fase 3: Configuración Híbrida (Usuario: `hadoop_user`)

### 4.1. Configuración de Entorno (`conf/atlas-env.sh`)
Establecer la burbuja de Java 11:
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export MANAGE_LOCAL_HBASE=false
export MANAGE_LOCAL_SOLR=false
export ATLAS_SERVER_HEAP="-Xms2g -Xmx4g"
```

### 4.2. Propiedades Principales (`conf/atlas-application.properties`)
Configuración estricta para integración de red:
```properties
atlas.graph.storage.backend=hbase2
atlas.graph.storage.hostname=localhost
atlas.graph.storage.port=2181

atlas.graph.index.search.backend=solr
atlas.graph.index.search.solr.mode=cloud
atlas.graph.index.search.solr.zookeeper-url=localhost:2181

atlas.notification.embedded=false
atlas.kafka.bootstrap.servers=localhost:9092

atlas.enableTLS=false
atlas.server.http.port=21000
```

---

## 5. Fase 4: Limpieza de Conflictos (Critical Fixes)
Al intentar arrancar la primera vez, el archivo `atlas.war` extrae librerías conflictivas en el classpath causando `StackOverflowError` y desvío de logs. 

Ejecutar estos comandos de saneamiento en el directorio raíz de Atlas:
```bash
# 1. Eliminar duplicidad de SLF4J / Log4J
find /usr/local/atlas -name "slf4j-reload4j*.jar" -exec mv {} {}.bak \;
find /usr/local/atlas -name "log4j-over-slf4j*.jar" -exec mv {} {}.bak \;
find /usr/local/atlas -name "log4j-slf4j-impl*.jar" -exec mv {} {}.bak \;

# 2. Eliminar contaminación de logs de pruebas
mv /usr/local/atlas/server/webapp/atlas/WEB-INF/lib/atlas-testtools-2.4.0.jar /usr/local/atlas/server/webapp/atlas/WEB-INF/lib/atlas-testtools-2.4.0.jar.bak
```

---

## 6. Fase 5: Despliegue del Servicio Systemd
El proceso de Setup inicial de Atlas requiere más de 5 minutos, provocando un error de timeout. Se configura a 15 minutos (900s).

Archivo: `/etc/systemd/system/atlas.service`
```ini
[Unit]
Description=Apache Atlas Metadata Service
After=network.target hbase.service solr.service kafka.service zookeeper.service

[Service]
Type=forking
User=hadoop_user
Group=hadoop_group
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
WorkingDirectory=/usr/local/atlas
ExecStart=/usr/bin/python3 /usr/local/atlas/bin/atlas_start.py
ExecStop=/usr/bin/python3 /usr/local/atlas/bin/atlas_stop.py
TimeoutStartSec=900
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Arranque Final:
```bash
sudo systemctl daemon-reload
sudo systemctl enable atlas.service
sudo systemctl start atlas.service

# Monitorear:
sudo journalctl -u atlas.service -f
```

## 7. Verificación del Sistema
Una vez completado el *Setup*, Atlas publicará su API e interfaz:
* **API Check:** `curl -u admin:admin http://localhost:21000/api/atlas/v2/types/typedefs/headers`
* **Web UI:** `http://<IP-SERVIDOR>:21000` (Credenciales por defecto: `admin` / `admin`).

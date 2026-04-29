# Guía Completa: Apache Solr 9 + Apache Atlas 2.4.0
### Solr en Java 17 — Atlas en burbuja Java 11

---

## Prerequisitos
- Hadoop, HBase, Kafka, ZooKeeper corriendo como systemd en Java 17
- Usuario de servicio: `hadoop_user:hadoop_group`
- Java 17 en: `/usr/lib/jvm/jdk-17.0.2`
- Archivos descargados en `/tmp`: `apache-atlas-2.4.0-sources.tar.gz`, `apache-maven-3.9.6-bin.tar.gz`

---

# PARTE 1: Apache Solr 9.7.0 (Java 17)

## Como uitga (sudo)

### Paso 1: Descargar e instalar Solr

```bash
cd /tmp
wget https://archive.apache.org/dist/solr/solr/9.7.0/solr-9.7.0.tgz
sudo tar -xzf solr-9.7.0.tgz -C /usr/local/
sudo mv /usr/local/solr-9.7.0 /usr/local/solr
sudo useradd -r -m -s /bin/bash -d /home/solr solr
sudo chown -R solr:solr /usr/local/solr
```

### Paso 2: Servicio systemd para Solr

```bash
sudo nano /etc/systemd/system/solr.service
```

```ini
[Unit]
Description=Apache Solr 9 SolrCloud
After=network.target zookeeper.service
Requires=zookeeper.service

[Service]
Type=forking
User=solr
Group=solr
Environment="JAVA_HOME=/usr/lib/jvm/jdk-17.0.2"
Environment="PATH=/usr/lib/jvm/jdk-17.0.2/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"
ExecStart=/usr/local/solr/bin/solr start -c -z localhost:2181 -p 8983 -noprompt
ExecStop=/usr/local/solr/bin/solr stop -p 8983
LimitNOFILE=65000
LimitNPROC=65000
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable solr.service
sudo systemctl start solr.service
sleep 20
sudo systemctl status solr.service
```

### Paso 3: Crear colecciones requeridas por Atlas

```bash
sudo su - solr -s /bin/bash
export JAVA_HOME=/usr/lib/jvm/jdk-17.0.2

/usr/local/solr/bin/solr create -c vertex_index -shards 1 -replicationFactor 1
/usr/local/solr/bin/solr create -c edge_index -shards 1 -replicationFactor 1
/usr/local/solr/bin/solr create -c fulltext_index -shards 1 -replicationFactor 1

# Verificar las 3 colecciones
curl http://localhost:8983/solr/admin/collections?action=LIST
exit
```

---

# PARTE 2: Apache Atlas 2.4.0 (Burbuja Java 11)

## Como uitga (sudo)

### Paso 4: Instalar Java 11 y Maven

```bash
sudo apt update
sudo apt install openjdk-11-jdk -y
/usr/lib/jvm/java-11-openjdk-amd64/bin/java -version

cd /tmp
wget https://archive.apache.org/dist/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
tar -xzf apache-maven-3.9.6-bin.tar.gz
sudo mv apache-maven-3.9.6 /usr/local/maven
sudo chown -R hadoop_user:hadoop_group /usr/local/maven
```

### Paso 5: Descargar fuentes de Atlas

```bash
cd /tmp
wget https://archive.apache.org/dist/atlas/2.4.0/apache-atlas-2.4.0-sources.tar.gz
tar -xzf apache-atlas-2.4.0-sources.tar.gz
sudo chown -R hadoop_user:hadoop_group /tmp/apache-atlas-sources-2.4.0
```

---

## Como hadoop_user — Compilación

```bash
sudo su - hadoop_user
```

### Paso 6: Variables de entorno

```bash
nano ~/.bashrc
```

Añadir al final:

```bash
export MAVEN_HOME=/usr/local/maven
export PATH=$PATH:$MAVEN_HOME/bin
export ATLAS_HOME=/usr/local/atlas
export PATH=$PATH:$ATLAS_HOME/bin
```

```bash
source ~/.bashrc
mvn -version  # debe mostrar Maven 3.9.6
```

### Paso 7: Parchear enunciate en atlas-webapp/pom.xml

```bash
cd /tmp/apache-atlas-sources-2.4.0
nano atlas-webapp/pom.xml
```

Busca con `Ctrl+W` el texto `enunciate-maven-plugin`. Dentro de ese bloque `<plugin>` agrega `<configuration><skip>true</skip></configuration>`:

```xml
<plugin>
    <groupId>com.webcohesion.enunciate</groupId>
    <artifactId>enunciate-maven-plugin</artifactId>
    <configuration>
        <skip>true</skip>
    </configuration>
</plugin>
```

Guarda con `Ctrl+O`, `Enter`, `Ctrl+X`.

### Paso 8: Compilar Atlas con Java 11

```bash
cd /tmp/apache-atlas-sources-2.4.0

export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
export MAVEN_OPTS="-Xms2g -Xmx4g -XX:MaxMetaspaceSize=1g"

mvn clean package -DskipTests \
  -Pdist \
  -Dmaven.javadoc.skip=true \
  -Drat.skip=true \
  -Dsort.skip=true \
  2>&1 | tee /tmp/atlas-build.log
```

Monitorea en otra terminal:

```bash
tail -f /tmp/atlas-build.log | grep -E "BUILD|ERROR|FAILURE|SUCCESS|atlas-webapp|Distribution"
```

Verificar al terminar — el WAR debe pesar más de 200MB:

```bash
find /tmp/apache-atlas-sources-2.4.0 -name "atlas.war" -ls
ls -lh /tmp/apache-atlas-sources-2.4.0/distro/target/apache-atlas-2.4.0-server.tar.gz
```

```bash
exit  # volver a uitga
```

---

## Como uitga — Instalación

### Paso 9: Instalar binarios

```bash
sudo tar -xzf /tmp/apache-atlas-sources-2.4.0/distro/target/apache-atlas-2.4.0-server.tar.gz -C /usr/local/
sudo mv /usr/local/apache-atlas-2.4.0 /usr/local/atlas
sudo chown -R hadoop_user:hadoop_group /usr/local/atlas
```

### Paso 10: Crear directorios y permisos

```bash
sudo mkdir -p /usr/local/atlas/data/kafka
sudo mkdir -p /usr/local/atlas/logs
sudo chown -R hadoop_user:hadoop_group /usr/local/atlas/data/
sudo chown -R hadoop_user:hadoop_group /usr/local/atlas/logs/
```

---

## Como hadoop_user — Configuración

```bash
sudo su - hadoop_user
```

### Paso 11: atlas-env.sh

```bash
nano /usr/local/atlas/conf/atlas-env.sh
```

Reemplaza todo el contenido:

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export MANAGE_LOCAL_HBASE=false
export MANAGE_LOCAL_SOLR=false
export HBASE_CONF_DIR=/usr/local/hbase/conf
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
export ATLAS_SERVER_HEAP="-Xms2g -Xmx4g -XX:MaxMetaspaceSize=512m"
```

### Paso 12: atlas-application.properties

```bash
nano /usr/local/atlas/conf/atlas-application.properties
```

Reemplaza todo el contenido:

```properties
# Graph Storage — HBase externo
atlas.graph.storage.backend=hbase2
atlas.graph.storage.hostname=localhost
atlas.graph.storage.port=2181
atlas.graph.storage.hbase.table=apache_atlas_janus
atlas.graph.storage.hbase.regions-per-server=1
atlas.graph.storage.hbase.zookeeper.quorum=localhost
atlas.graph.storage.hbase.zookeeper.property.clientPort=2181
atlas.graph.storage.hbase.zookeeper.znode.parent=/hbase

# Índice — Solr 9 externo en modo cloud
atlas.graph.index.search.backend=solr
atlas.graph.index.search.solr.mode=cloud
atlas.graph.index.search.solr.zookeeper-url=localhost:2181
atlas.graph.index.search.solr.zookeeper-connect-timeout=60000
atlas.graph.index.search.solr.zookeeper-session-timeout=60000
atlas.graph.index.search.solr.wait-searcher=true

# Kafka externo
atlas.notification.embedded=false
atlas.kafka.bootstrap.servers=localhost:9092
atlas.kafka.zookeeper.connect=localhost:2181
atlas.kafka.zookeeper.session.timeout.ms=400
atlas.kafka.zookeeper.connection.timeout.ms=200
atlas.kafka.zookeeper.sync.time.ms=20
atlas.kafka.hook.group.id=atlas
atlas.kafka.data=/usr/local/atlas/data/kafka

# Auditoría
atlas.audit.hbase.tablename=apache_atlas_entity_audit
atlas.audit.hbase.zookeeper.quorum=localhost
atlas.audit.zookeeper.session.timeout.ms=1000

# Servidor HTTP
atlas.enableTLS=false
atlas.server.http.port=21000
atlas.server.bind.address=0.0.0.0

# Autenticación
atlas.authentication.method.file=true
atlas.authentication.method.file.filename=${sys:atlas.home}/conf/users-credentials.properties
atlas.auth.policy.file=${sys:atlas.home}/conf/atlas-simple-authz-policy.json
atlas.authorizer.impl=simple
```

### Paso 13: Limpiar conflictos de librerías (crítico)

Este paso es el más importante. Atlas extrae jars en runtime que crean conflictos SLF4J. Hay que eliminar exactamente estos tres — pero **conservar logback-classic** porque Atlas lo necesita:

```bash
# Renombrar los dos conflictivos
mv /usr/local/atlas/server/webapp/atlas/WEB-INF/lib/slf4j-reload4j-2.0.16.jar \
   /usr/local/atlas/server/webapp/atlas/WEB-INF/lib/slf4j-reload4j-2.0.16.jar.bak

mv /usr/local/atlas/server/webapp/atlas/WEB-INF/lib/log4j-over-slf4j-2.0.16.jar \
   /usr/local/atlas/server/webapp/atlas/WEB-INF/lib/log4j-over-slf4j-2.0.16.jar.bak

# Renombrar jar de tests que desvía el logging
find /usr/local/atlas -name "atlas-testtools*.jar" -not -name "*.bak" \
  -exec mv {} {}.bak \;

# Verificar — estas búsquedas deben devolver vacío
echo "=== Deben estar vacíos ==="
find /usr/local/atlas -name "slf4j-reload4j*.jar" -not -name "*.bak"
find /usr/local/atlas -name "log4j-over-slf4j*.jar" -not -name "*.bak"
find /usr/local/atlas -name "atlas-testtools*.jar" -not -name "*.bak"

# logback-classic DEBE seguir existiendo
echo "=== logback-classic (debe existir) ==="
find /usr/local/atlas -name "logback-classic*.jar" -not -name "*.bak"

exit  # volver a uitga
```

---

## Como uitga — Systemd

### Paso 14: Servicio atlas.service

```bash
sudo nano /etc/systemd/system/atlas.service
```

```ini
[Unit]
Description=Apache Atlas Metadata Service
After=network.target zookeeper.service hbase.service kafka.service solr.service
Requires=zookeeper.service hbase.service kafka.service solr.service

[Service]
Type=forking
User=hadoop_user
Group=hadoop_group
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
Environment="PATH=/usr/lib/jvm/java-11-openjdk-amd64/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="ATLAS_HOME=/usr/local/atlas"
Environment="HBASE_CONF_DIR=/usr/local/hbase/conf"
Environment="HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop"
WorkingDirectory=/usr/local/atlas
ExecStart=/usr/bin/python3 /usr/local/atlas/bin/atlas_start.py
ExecStop=/usr/bin/python3 /usr/local/atlas/bin/atlas_stop.py
TimeoutStartSec=900
Restart=on-failure
RestartSec=15s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable atlas.service
```

---

## Arranque y verificación

### Paso 15: Limpiar logs anteriores e iniciar

```bash
sudo rm -f /usr/local/atlas/logs/*
sudo systemctl start atlas.service
```

### Paso 16: Monitorear

La primera vez tarda 5-8 minutos creando tablas en HBase:

```bash
# En una terminal — ver logs en tiempo real
sudo journalctl -u atlas.service -f
```

En otra terminal cuando aparezca `application.log`:

```bash
tail -f /usr/local/atlas/logs/application.log
```

Espera hasta ver: `Apache Atlas Server started!!!`

### Paso 17: Verificación final

```bash
ss -tlnp | grep 21000
curl -u admin:admin http://localhost:21000/api/atlas/v2/types/typedefs/headers
sudo ufw allow 21000/tcp
```

Accede desde navegador: `http://IP_DEL_SERVIDOR:21000`
Credenciales: `admin` / `admin`

---

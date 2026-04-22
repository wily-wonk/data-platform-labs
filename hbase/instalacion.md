# Guía de Instalación: Arquitectura Desacoplada de HBase y ZooKeeper

Esta guía detalla la instalación y configuración de Apache ZooKeeper (Standalone) y Apache HBase sobre un clúster de Hadoop existente. Se implementa una arquitectura en la que ZooKeeper opera como un servicio independiente, desactivando la gestión embebida de HBase para garantizar una mayor estabilidad y preparación para entornos productivos en el servidor Debian.



## FASE 1: Instalación de Apache ZooKeeper Independiente

### 1. Descarga y Ubicación del Binario
**Usuario requerido:** Administrador (server con permisos sudo)
Se utiliza la versión estable 3.8.6, la cual cuenta con soporte nativo para la versión de Java 17.0.2 instalada en el clúster.

```bash
cd /tmp
wget https://archive.apache.org/dist/zookeeper/zookeeper-3.8.6/apache-zookeeper-3.8.6-bin.tar.gz
tar -xzf apache-zookeeper-3.8.6-bin.tar.gz
sudo mv apache-zookeeper-3.8.6-bin /usr/local/zookeeper
```

### 2. Creación de Directorio de Datos y Permisos
**Usuario requerido:** Administrador (server)

```bash
sudo mkdir -p /usr/local/zookeeper/data
sudo chown -R hadoop_user:hadoop_group /usr/local/zookeeper
```

### 3. Configuración de Variables de Entorno
**Usuario requerido:** hadoop_user
Iniciamos sesión con el usuario de servicio y agregamos las rutas al entorno:

```bash
sudo su - hadoop_user
nano ~/.bashrc
```

Agregar al final del archivo:

```bash
# Apache ZooKeeper
export ZOOKEEPER_HOME=/usr/local/zookeeper
export PATH=$PATH:$ZOOKEEPER_HOME/bin

# Apache HBase
export HBASE_HOME=/usr/local/hbase
export PATH=$PATH:$HBASE_HOME/bin
```

Aplicar los cambios:
```bash
source ~/.bashrc
```

### 4. Configuración del Nodo (zoo.cfg)
**Usuario requerido:** hadoop_user
Asignamos el ID del nodo y configuramos los puertos:

```bash
echo "1" > /usr/local/zookeeper/data/myid
cp /usr/local/zookeeper/conf/zoo_sample.cfg /usr/local/zookeeper/conf/zoo.cfg
nano /usr/local/zookeeper/conf/zoo.cfg
```

Modificar la propiedad `dataDir` para que coincida con nuestro directorio:
```ini
dataDir=/usr/local/zookeeper/data
clientPort=2181
```

### 5. Creación del Servicio Systemd para ZooKeeper
**Usuario requerido:** Administrador (server)
Salimos del usuario `hadoop_user` (ejecutando `exit`) y creamos el servicio:

```bash
sudo nano /etc/systemd/system/zookeeper.service
```

Contenido del archivo (con la ruta específica del JDK 17.0.2):
```ini
[Unit]
Description=Apache ZooKeeper Standalone
After=network.target

[Service]
Type=forking
User=hadoop_user
Group=hadoop_group
Environment=JAVA_HOME=/usr/lib/jvm/jdk-17.0.2
ExecStart=/usr/local/zookeeper/bin/zkServer.sh start
ExecStop=/usr/local/zookeeper/bin/zkServer.sh stop
ExecReload=/usr/local/zookeeper/bin/zkServer.sh restart
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Habilitar e iniciar el servicio:
```bash
sudo systemctl daemon-reload
sudo systemctl enable zookeeper.service
sudo systemctl start zookeeper.service
```

---

## FASE 2: Instalación de Apache HBase

### 6. Descarga y Ubicación del Binario
**Usuario requerido:** Administrador (server)
Se debe utilizar obligatoriamente el binario compilado para Hadoop 3.x.

```bash
cd /tmp
wget https://archive.apache.org/dist/hbase/2.5.13/hbase-2.5.13-hadoop3-bin.tar.gz
tar -xzf hbase-2.5.13-hadoop3-bin.tar.gz
sudo mv hbase-2.5.13-hadoop3 /usr/local/hbase
sudo chown -R hadoop_user:hadoop_group /usr/local/hbase
```

### 7. Configuración del Entorno de HBase (hbase-env.sh)
**Usuario requerido:** hadoop_user

```bash
sudo su - hadoop_user
nano /usr/local/hbase/conf/hbase-env.sh
```

Configuración crítica para desactivar el ZooKeeper interno y permitir la compatibilidad de módulos con Java 17:
```bash
export JAVA_HOME=/usr/lib/jvm/jdk-17.0.2

# Desactivar gestión interna de ZooKeeper
export HBASE_MANAGES_ZK=false

# Opciones para prevenir InaccessibleObjectException en Java 17
export HBASE_OPTS="$HBASE_OPTS \
  --add-opens java.base/java.io=ALL-UNNAMED \
  --add-opens java.base/java.nio=ALL-UNNAMED \
  --add-opens java.base/sun.nio.ch=ALL-UNNAMED \
  --add-opens java.base/java.lang=ALL-UNNAMED \
  --add-opens java.base/java.util=ALL-UNNAMED \
  --add-opens java.base/java.util.concurrent=ALL-UNNAMED \
  --add-opens java.base/java.net=ALL-UNNAMED"
```

### 8. Configuración de Conexión (hbase-site.xml)
**Usuario requerido:** hadoop_user

```bash
nano /usr/local/hbase/conf/hbase-site.xml
```

Archivo original Por defecto
```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
-->
<configuration>
  <!--
    The following properties are set for running HBase as a single process on a
    developer workstation. With this configuration, HBase is running in
    "stand-alone" mode and without a distributed file system. In this mode, and
    without further configuration, HBase and ZooKeeper data are stored on the
    local filesystem, in a path under the value configured for `hbase.tmp.dir`.
    This value is overridden from its default value of `/tmp` because many
    systems clean `/tmp` on a regular basis. Instead, it points to a path within
    this HBase installation directory.

    Running against the `LocalFileSystem`, as opposed to a distributed
    filesystem, runs the risk of data integrity issues and data loss. Normally
    HBase will refuse to run in such an environment. Setting
    `hbase.unsafe.stream.capability.enforce` to `false` overrides this behavior,
    permitting operation. This configuration is for the developer workstation
    only and __should not be used in production!__

    See also https://hbase.apache.org/book.html#standalone_dist
  -->
  <property>
    <name>hbase.cluster.distributed</name>
    <value>false</value>
  </property>
  <property>
    <name>hbase.tmp.dir</name>
    <value>./tmp</value>
  </property>
  <property>
    <name>hbase.unsafe.stream.capability.enforce</name>
    <value>false</value>
  </property>
</configuration>
```
Insertar o reemplazar todo la siguiente configuración (reemplazar `ip-server` por el hostname del servidor Debian definido en `/etc/hosts`):
```xml
<configuration>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://ip-server:9000/hbase</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>ip-server</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
  </property>
</configuration>
```

Configuracion que tenia por defecto
```xml
```xml
<configuration>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://ip-server:9000/hbase</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>ip-server</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
  </property>
</configuration>
```

```

### 9. Integración con HDFS
**Usuario requerido:** hadoop_user
Enlazar los archivos de configuración de Hadoop y crear el directorio raíz en HDFS:

```bash
ln -s /usr/local/hadoop/etc/hadoop/core-site.xml /usr/local/hbase/conf/core-site.xml
ln -s /usr/local/hadoop/etc/hadoop/hdfs-site.xml /usr/local/hbase/conf/hdfs-site.xml
hdfs dfs -mkdir -p /hbase
hdfs dfs -chown -R hadoop_user:hadoop_group /hbase
```

### 10. Creación del Servicio Systemd para HBase
**Usuario requerido:** Administrador (server)
Salimos del usuario `hadoop_user` (`exit`) y creamos el servicio:

```bash
sudo nano /etc/systemd/system/hbase.service
```

El bloque requiere que HDFS y ZooKeeper arranquen antes que HBase:
```ini
[Unit]
Description=Apache HBase
After=network.target hadoop-namenode.service hadoop-datanode.service zookeeper.service
Wants=hadoop-namenode.service zookeeper.service

[Service]
Type=forking
User=hadoop_user
Group=hadoop_group
Environment=JAVA_HOME=/usr/lib/jvm/jdk-17.0.2
Environment=HBASE_HOME=/usr/local/hbase
ExecStart=/usr/local/hbase/bin/start-hbase.sh
ExecStop=/usr/local/hbase/bin/stop-hbase.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Habilitar e iniciar el servicio:
```bash
sudo systemctl daemon-reload
sudo systemctl enable hbase.service
sudo systemctl start hbase.service
```

---

## FASE 3: Verificación del Clúster

Para validar que la arquitectura está operando correctamente, ejecutamos `jps` como `hadoop_user`:

```bash
sudo su - hadoop_user
jps
```

**Salida Esperada:** Debe aparecer el proceso `QuorumPeerMain` en lugar de `HQuorumPeer`, lo que confirma que ZooKeeper está operando de forma autónoma junto con los servicios de HBase.

```text
3731 HMaster
3840 HRegionServer
1254 QuorumPeerMain
```


# Instalación Apache Hive 4.0.0 — Compatible con Java 17

**Prerrequisito:** Hadoop corriendo como daemon (`systemctl status hadoop-dfs hadoop-yarn` en verde).
**IP Servidor:** 192.168.0.8
**Ruta de Java:** `/usr/lib/jvm/jdk-17.0.2`

## Fase 1: Como usuario administrador (`sudo`)

### 1. Descarga e instalación de binarios
```bash
cd /tmp
wget https://archive.apache.org/dist/hive/hive-4.0.0/apache-hive-4.0.0-bin.tar.gz
tar -xzvf apache-hive-4.0.0-bin.tar.gz
sudo mv apache-hive-4.0.0-bin /usr/local/hive
sudo chown -R hadoop_user:hadoop_group /usr/local/hive
```

### 2. Instalación de PostgreSQL (Metastore)
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Crear base de datos y usuario para Hive
sudo -u postgres psql -c "CREATE DATABASE metastore_db;"
sudo -u postgres psql -c "CREATE USER hiveuser WITH PASSWORD 'hivepassword';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE metastore_db TO hiveuser;"
sudo -u postgres psql -c "ALTER DATABASE metastore_db OWNER TO hiveuser;"
```

### 3. Driver JDBC de PostgreSQL
```bash
wget -P /tmp https://jdbc.postgresql.org/download/postgresql-42.7.2.jar
sudo cp /tmp/postgresql-42.7.2.jar /usr/local/hive/lib/
sudo chown hadoop_user:hadoop_group /usr/local/hive/lib/postgresql-42.7.2.jar
```

### 4. Fix de impersonación en Hadoop (crítico)
```bash
sudo nano /usr/local/hadoop/etc/hadoop/core-site.xml
```
*Reemplaza el contenido con:*
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.0.8:9000</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hadoop_user.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hadoop_user.groups</name>
        <value>*</value>
    </property>
</configuration>
```

*Reiniciar Hadoop para aplicar el cambio:*
```bash
sudo systemctl restart hadoop-dfs
sudo systemctl restart hadoop-yarn
```

---

## Fase 2: Como `hadoop_user`
```bash
sudo su - hadoop_user
```

### 5. Variables de entorno
```bash
nano ~/.bashrc
```
*Añadir al final:*
```bash
# Apache Hive
export HIVE_HOME=/usr/local/hive
export PATH=$PATH:$HIVE_HOME/bin
```
*Aplicar:*
```bash
source ~/.bashrc
```

### 6. Parche Java 17 — `hive-env.sh`
```bash
cp $HIVE_HOME/conf/hive-env.sh.template $HIVE_HOME/conf/hive-env.sh
cat << 'EOF' > /usr/local/hive/conf/hive-env.sh
# Configuración específica para Java 17
export JAVA_HOME=/usr/lib/jvm/jdk-17.0.2
export HADOOP_HOME=/usr/local/hadoop

# Flags para abrir módulos encapsulados de Java 17
JAVA17_OPENS="--add-opens java.base/java.lang=ALL-UNNAMED \
  --add-opens java.base/java.net=ALL-UNNAMED \
  --add-opens java.base/java.nio=ALL-UNNAMED \
  --add-opens java.base/java.util=ALL-UNNAMED \
  --add-opens java.base/java.util.concurrent=ALL-UNNAMED \
  --add-opens java.base/sun.nio.ch=ALL-UNNAMED \
  --add-opens java.security.jgss/sun.security.krb5=ALL-UNNAMED"

# Inyectar flags exclusivamente al cliente y servidor de Hadoop/JVM
export HADOOP_CLIENT_OPTS="$JAVA17_OPENS"
export HIVE_SERVER2_HADOOP_OPTS="$JAVA17_OPENS"

# ESTA ES LA VACUNA: Prevenir que Hive intente leer flags de JVM directamente
export HIVE_OPTS=""
EOF
```

### 7. Configuración del Metastore — `hive-site.xml`
```bash
cat << 'EOF' > /usr/local/hive/conf/hive-site.xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:postgresql://localhost:5432/metastore_db</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>org.postgresql.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hiveuser</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hivepassword</value>
    </property>
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
    <property>
        <name>datanucleus.schema.autoCreateAll</name>
        <value>true</value>
    </property>
    <property>
        <name>hive.server2.enable.doAs</name>
        <value>true</value>
    </property>
</configuration>
EOF
```

### 8. Preparar HDFS e inicializar esquema
```bash
# Crear directorios en HDFS
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -mkdir -p /tmp
hdfs dfs -chmod g+w /user/hive/warehouse
hdfs dfs -chmod g+w /tmp

# Inicializar tablas del metastore en PostgreSQL
schematool -dbType postgres -initSchema
```
*(Debe terminar con: `Initialization script completed`)*.

---

## Fase 3: Como administrador (`sudo`) — Daemon de Hive
```bash
exit  # salir de hadoop_user
```

### 9. Servicio HiveServer2 con Systemd

**El misterio del "Systemd Bubble" (Burbuja de Systemd)**
Cuando ejecutas comandos manualmente, el usuario lee `~/.bashrc`. Systemd arranca servicios en una "burbuja" aislada sin leer perfiles. La solución es inyectar el `PATH` completo directamente en el servicio.

**Paso 1: Crear el archivo del servicio**
```bash
sudo nano /etc/systemd/system/hive-server2.service
```
*Contenido del servicio:*
```ini
[Unit]
Description=Apache HiveServer2
After=hadoop-dfs.service hadoop-yarn.service postgresql.service
Requires=hadoop-dfs.service hadoop-yarn.service postgresql.service

[Service]
Type=simple
User=hadoop_user
Group=hadoop_group
Environment="HIVE_HOME=/usr/local/hive"
Environment="HADOOP_HOME=/usr/local/hadoop"
Environment="JAVA_HOME=/usr/lib/jvm/jdk-17.0.2"
Environment="PATH=/usr/lib/jvm/jdk-17.0.2/bin:/usr/local/hadoop/bin:/usr/local/hive/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/usr/local/hive/bin/hive --service hiveserver2
Restart=on-failure
RestartSec=10s
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Paso 2: Apertura de Puerto y Arranque**
```bash
# Abrir puerto de conexión
sudo ufw allow 10000/tcp
sudo ufw reload

# Recargar demonio y arrancar servicio
sudo systemctl daemon-reload
sudo systemctl enable hive-server2.service
sudo systemctl restart hive-server2.service
```

**Paso 3: Validación**
Verifica el estado; debería decir `active (running)`.
```bash
sudo systemctl status hive-server2.service
```
*(Espera 1 o 2 minutos para que el motor levante y luego valida el puerto)*:
```bash
ss -tlnp | sudo grep 10000
```

---

## Fase 4: Validación final
```bash
# Entrar a la cuenta de servicio
sudo su - hadoop_user

# Conectar al cliente Beeline
beeline -u "jdbc:hive2://localhost:10000" -n hadoop_user
```

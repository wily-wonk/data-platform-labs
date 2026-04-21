# Guía de Instalación: Apache Kafka (KRaft Mode) + Kafka Connect (Debezium)

Esta documentación detalla la implementación de un ecosistema de mensajería distribuida utilizando el protocolo **KRaft**, eliminando la dependencia de ZooKeeper. La arquitectura incluye **Kafka Connect** para captura de datos (CDC) y **Kafka UI** para la administración visual.

**Entorno:** Debian GNU/Linux 10 (buster)
**IP del Servidor:** 192.168.0.8 (Ajustar según `ip a`)
**Java:** OpenJDK 17.0.2 (Manual en `/usr/lib/jvm/jdk-17.0.2`)

---

## FASE 1: Preparación del Entorno

### 1. Instalación de Dependencias
Aseguramos las herramientas de red y descompresión.
```bash
sudo apt update && sudo apt install curl wget tar rsync -y
```

### 2. Cuenta de Servicio Kafka
Se crea un usuario de sistema dedicado para aislar los procesos del clúster.
```bash
sudo groupadd kafka
sudo useradd -r -m -d /opt/kafka -g kafka -s /bin/false kafka

# Agregar el administrador al grupo para tareas de mantenimiento
sudo usermod -aG kafka $USER
```

## FASE 2: Instalación de Binarios y Plugins

### 3. Despliegue de Kafka
```bash
cd /tmp
wget https://archive.apache.org/dist/kafka/3.7.2/kafka_2.13-3.7.2.tgz
tar -xzf kafka_2.13-3.7.2.tgz
sudo mv kafka_2.13-3.7.2 /opt/kafka
```

### 4. Kafka UI (Provectus)
Interfaz gráfica para la gestión de tópicos y consumidores.
```bash
sudo curl -L https://github.com/provectus/kafka-ui/releases/download/v0.7.2/kafka-ui-api-v0.7.2.jar -o /opt/kafka/kafka-ui.jar
```

### 5. Plugin Debezium (PostgreSQL)
Conector para la captura de cambios en bases de datos.
```bash
cd /tmp
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/2.5.4.Final/debezium-connector-postgres-2.5.4.Final-plugin.tar.gz
sudo mkdir -p /opt/kafka/plugins
sudo tar -xzf debezium-connector-postgres-2.5.4.Final-plugin.tar.gz -C /opt/kafka/plugins

# Asignar propiedad al usuario de servicio
sudo chown -R kafka:kafka /opt/kafka
```



## FASE 3: Configuración de Red y Persistencia

### 6. Configuración de KRaft (`server.properties`)
```bash
sudo nano /opt/kafka/config/kraft/server.properties
```
*Modificar las siguientes propiedades:*
```properties
# Ruta persistente para logs y metadatos
log.dirs=/opt/kafka/data

# Configuración de red (listeners)
listeners=PLAINTEXT://0.0.0.0:9092,CONTROLLER://localhost:9093
advertised.listeners=PLAINTEXT://192.168.0.8:9092
```

### 7. Kafka Connect Distributed
```bash
sudo nano /opt/kafka/config/connect-distributed.properties
```
*Configurar el acceso a plugins y brokers:*
```properties
bootstrap.servers=localhost:9092
plugin.path=/opt/kafka/plugins
listeners=HTTP://0.0.0.0:8083
```



## FASE 4: Almacenamiento y Formateo (Corrección de JAVA_HOME)

Dado que Java se instaló manualmente, es obligatorio inyectar la variable `JAVA_HOME` para que los scripts de formateo localicen el binario.

### 8. Formateo del Clúster
```bash
# 1. Crear directorio físico de datos
sudo mkdir -p /opt/kafka/data
sudo chown -R kafka:kafka /opt/kafka

# 2. Generar UUID del clúster (Inyectando ruta de Java)
cd /opt/kafka
KAFKA_CLUSTER_ID=$(sudo -u kafka env JAVA_HOME=/usr/lib/jvm/jdk-17.0.2 bin/kafka-storage.sh random-uuid)

# 3. Formatear directorios de almacenamiento
sudo -u kafka env JAVA_HOME=/usr/lib/jvm/jdk-17.0.2 bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
```

## FASE 5: Automatización con Systemd

### 9. Servicio Principal: `kafka.service`
```bash
sudo nano /etc/systemd/system/kafka.service
```
```ini
[Unit]
Description=Apache Kafka Server (KRaft)
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka
Environment="JAVA_HOME=/usr/lib/jvm/jdk-17.0.2"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

### 10. Gestión Visual: `kafka-ui.service`
```bash
sudo nano /etc/systemd/system/kafka-ui.service
```
```ini
[Unit]
Description=Kafka UI (Provectus)
After=network.target kafka.service

[Service]
Type=simple
User=kafka
Group=kafka
Environment="AUTH_TYPE=disabled"
Environment="KAFKA_CLUSTERS_0_NAME=DataLake-Kafka"
Environment="KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=localhost:9092"
Environment="SERVER_PORT=8070"
# Se utiliza la ruta absoluta al binario de Java
ExecStart=/usr/lib/jvm/jdk-17.0.2/bin/java -jar /opt/kafka/kafka-ui.jar
Restart=always

[Install]
WantedBy=multi-user.target
```

### 11. Motor de Conexión: `kafka-connect.service`
```bash
sudo nano /etc/systemd/system/kafka-connect.service
```
```ini
[Unit]
Description=Apache Kafka Connect Distributed (Debezium)
After=network.target kafka.service
Requires=kafka.service

[Service]
Type=simple
User=kafka
Group=kafka
Environment="JAVA_HOME=/usr/lib/jvm/jdk-17.0.2"
ExecStart=/opt/kafka/bin/connect-distributed.sh /opt/kafka/config/connect-distributed.properties
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

## FASE 6: Seguridad de Red (Firewall)
Se exponen los puertos necesarios para la operación del clúster.
```bash
sudo ufw allow 8070/tcp   # Kafka UI
sudo ufw allow 8083/tcp   # Kafka Connect API
sudo ufw allow 9092/tcp   # Kafka Broker
sudo ufw reload
```

## FASE 7: Operación y Verificación

### 12. Arranque de Servicios
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kafka kafka-ui kafka-connect
```

### 13. Auditoría de Procesos
```bash
# Validar servicios activos
sudo systemctl status kafka kafka-ui kafka-connect

# Verificar API de Connect
curl http://localhost:8083/connectors
```

**Acceso Web:**
* **Kafka UI:** `http://192.168.0.8:8070`

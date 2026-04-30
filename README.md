
# Ecosistema

> Servidor: `GMLPSR014DB048` | Debian 10 | Java 17 (excepto Atlas con Java 11)

## Servicios Activos

| Servicio | Systemd Unit | Versión | Puerto | Usuario |
|----------|-------------|---------|--------|---------|
| **ZooKeeper** | `zookeeper.service` | 3.8.6 | 2181 | `hadoop_user` |
| **Kafka** | `kafka.service` | 3.7.2 | 9092 | `kafka` |
| **Kafka Connect** | `kafka-connect.service` | 3.7.2 (Debezium) | 8083 | `kafka` |
| **Kafka UI** | `kafka-ui.service` | — | 8080 | `kafka` |
| **Hadoop HDFS** | `hadoop-hdfs.service` | 3.4.0 | 9870, 8070, 9000 | `hadoop_user` |
| **Hadoop YARN** | `hadoop-yarn.service` | 3.4.0 | 8088 | `hadoop_user` |
| **HBase** | `hbase.service` | 2.5.13 | 16010, 16020, 16030 | `hadoop_user` |
| **Hive** | `hive-server2.service` | 4.0.0 | 10000, 10002 | `hadoop_user` |
| **Solr** | `solr.service` | 9.7.0 | 8983 | `solr` |
| **Atlas** | `atlas.service` | 2.4.0 | 21000 | `hadoop_user` |
| **NiFi** | `nifi.service` | 1.28.1 | 8443 (HTTPS) | `nifi` |
| **Airflow** | `airflow-webserver.service` `airflow-scheduler.service` | 2.6.3 | 8081 | `airflow` |
| **PostgreSQL** | `postgresql@11-main.service` | 11.22 | 5432 | `postgres` |
| **Apache HTTP** | `apache2.service` | 2.4.x | 80 | `root` |
| **NTP** | `ntp.service` | — | 123 | `root` |

## Mapa de Puertos

```
2181 - ZooKeeper
9092 - Kafka Broker
8083 - Kafka Connect REST API
8080 - Kafka UI
9870 - HDFS NameNode Web UI
8070 - HDFS NameNode HTTP
9000 - HDFS NameNode RPC
8088 - YARN ResourceManager Web UI
16010 - HBase Master Web UI
16020 - HBase RegionServer
16030 - HBase RegionServer Web UI
10000 - HiveServer2 Thrift
10002 - HiveServer2 Web UI
8983 - Solr 9 Cloud
21000 - Atlas Web UI
8443 - NiFi Web UI (HTTPS)
8081 - Airflow Webserver
5432 - PostgreSQL
80   - Apache HTTP Server
22   - SSH
```

## Arquitectura de Java

| Componente | Java | Ruta |
|-----------|------|------|
| ZooKeeper | 17 | `/usr/lib/jvm/jdk-17.0.2` |
| Kafka | 17 | `/usr/lib/jvm/jdk-17.0.2` |
| Hadoop | 17 | `/usr/lib/jvm/jdk-17.0.2` |
| HBase | 17 | `/usr/lib/jvm/jdk-17.0.2` |
| Hive | 17 | `/usr/lib/jvm/jdk-17.0.2` |
| Solr | 17 | `/usr/lib/jvm/jdk-17.0.2` |
| NiFi | 17 | `/usr/lib/jvm/jdk-17.0.2` |
| **Atlas** | **11** | `/usr/lib/jvm/java-11-openjdk-amd64` |

## Comandos Útiles

```bash
# Ver todos los servicios
systemctl list-units --type=service --state=running

# Reiniciar todo el ecosistema
sudo systemctl restart zookeeper kafka hadoop-hdfs hadoop-yarn hbase hive-server2 solr atlas nifi airflow-webserver airflow-scheduler

# Ver logs de un servicio
sudo journalctl -u <servicio> -f
```

---

##  URLs 

| Herramienta | URL | Puerto |
|------------|-----|--------|
| **NiFi** | `https://192.168.7.46:8443/nifi` | 8443 |
| **Atlas** | `http://192.168.7.46:21000` | 21000 |
| **Airflow** | `http://192.168.7.46:8081` | 8081 |
| **HDFS NameNode** | `http://192.168.7.46:9870` | 9870 |
| **YARN** | `http://192.168.7.46:8088` | 8088 |
| **HBase Master** | `http://192.168.7.46:16010` | 16010 |
| **HiveServer2** | `http://192.168.7.46:10002` | 10002 |
| **Solr** | `http://192.168.7.46:8983/solr` | 8983 |
| **Kafka UI** | `http://192.168.7.46:8070` | 8070 |

---
## Rutas de Configuración

| Componente | Archivo de Configuración | Usuario | Carpeta Principal |
|-----------|-------------------------|--------|-------------------|
| **ZooKeeper** | `/usr/local/zookeeper/conf/zoo.cfg` | `hadoop_user` | `/usr/local/zookeeper` |
| **Kafka** | `/opt/kafka/config/server.properties` | `kafka` | `/opt/kafka` |
| **Kafka Connect** | `/opt/kafka/config/connect-distributed.properties` | `kafka` | `/opt/kafka` |
| **Kafka UI** | `/opt/kafka/config/kafka-ui.yml`  | `kafka` | `/opt/kafka` |
| **Hadoop HDFS** | `/usr/local/hadoop/etc/hadoop/core-site.xml` | `hadoop_user` | `/usr/local/hadoop` |
| **Hadoop HDFS** | `/usr/local/hadoop/etc/hadoop/hdfs-site.xml` | `hadoop_user` | `/usr/local/hadoop` |
| **Hadoop YARN** | `/usr/local/hadoop/etc/hadoop/yarn-site.xml` | `hadoop_user` | `/usr/local/hadoop` |
| **HBase** | `/usr/local/hbase/conf/hbase-site.xml` | `hadoop_user` | `/usr/local/hbase` |
| **Hive** | `/usr/local/hive/conf/hive-site.xml` | `hadoop_user` | `/usr/local/hive` |
| **Solr** | `/usr/local/solr/bin/solr.in.sh` | `solr` | `/usr/local/solr` |
| **Atlas** | `/usr/local/atlas/conf/atlas-application.properties` | `hadoop_user` | `/usr/local/atlas` |
| **Atlas** | `/usr/local/atlas/conf/atlas-env.sh` | `hadoop_user` | `/usr/local/atlas` |
| **NiFi** | `/opt/nifi/conf/nifi.properties` | `nifi` | `/opt/nifi` |
| **Airflow** | `/usr/local/airflow/airflow.cfg` | `airflow` | `/usr/local/airflow` |
| **PostgreSQL** | `/etc/postgresql/11/main/postgresql.conf` | `postgres` | `/etc/postgresql/11/main` |
| **PostgreSQL** | `/etc/postgresql/11/main/pg_hba.conf` | `postgres` | `/etc/postgresql/11/main` |

## Rutas de JARs/Librerías

| Componente | Carpeta de Librerías |
|-----------|---------------------|
| **Kafka** | `/opt/kafka/libs/` |
| **Hadoop** | `/usr/local/hadoop/share/hadoop/` |
| **HBase** | `/usr/local/hbase/lib/` |
| **Hive** | `/usr/local/hive/lib/` |
| **Solr** | `/usr/local/solr/server/solr-webapp/webapp/WEB-INF/lib/` |
| **Atlas** | `/usr/local/atlas/server/webapp/atlas/WEB-INF/lib/` |
| **NiFi** | `/opt/nifi/lib/` |

## Rutas de Logs

| Componente | Carpeta de Logs | Comando para ver en vivo |
|-----------|----------------|--------------------------|
| **ZooKeeper** | `/usr/local/zookeeper/logs/` | `sudo journalctl -u zookeeper -f` |
| **Kafka** | `/opt/kafka/logs/` | `sudo journalctl -u kafka -f` |
| **Kafka Connect** | `/opt/kafka/logs/` | `sudo journalctl -u kafka-connect -f` |
| **Hadoop HDFS** | `/usr/local/hadoop/logs/` | `sudo journalctl -u hadoop-hdfs -f` |
| **HBase** | `/usr/local/hbase/logs/` | `sudo journalctl -u hbase -f` |
| **Hive** | `/usr/local/hive/logs/` | `sudo journalctl -u hive-server2 -f` |
| **Solr** | `/usr/local/solr/server/logs/` | `sudo journalctl -u solr -f` |
| **Atlas** | `/usr/local/atlas/logs/` | `sudo journalctl -u atlas -f` |
| **NiFi** | `/opt/nifi/logs/` | `sudo journalctl -u nifi -f` |
| **Airflow** | `/usr/local/airflow/logs/` | `sudo journalctl -u airflow-webserver -f` |
| **PostgreSQL** | `/var/log/postgresql/` | `sudo journalctl -u postgresql -f` |

## Usuarios del Sistema

| Usuario | Grupo | Servicios |
|--------|-------|-----------|
| `hadoop_user` | `hadoop_group` | ZooKeeper, Hadoop, HBase, Hive, Atlas |
| `kafka` | `kafka` | Kafka, Kafka Connect, Kafka UI |
| `solr` | `solr` | Solr |
| `nifi` | `nifi` | NiFi |
| `airflow` | `airflow` | Airflow |
| `postgres` | `postgres` | PostgreSQL |
| `uitga` | `uitga` | Administrador (sudo) |

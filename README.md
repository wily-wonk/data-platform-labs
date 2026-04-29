
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

# Despliegue de Apache Hadoop 3.4.0 (Single-Node) - Entorno Seguro

Esta documentación detalla el proceso de instalación, configuración y automatización del ecosistema Apache Hadoop. La arquitectura está diseñada aplicando el Principio de Mínimo Privilegio (PoLP), aislando los procesos distribuidos en una cuenta de servicio dedicada para garantizar la seguridad de la infraestructura y sentar las bases para la ingesta de datos.

**Entorno:** Debian GNU/Linux 10 (buster)
**Rol del Nodo:** Single-Node Cluster (NameNode, DataNode, ResourceManager, NodeManager)
**Versión de Hadoop:** 3.4.0
**Versión de Java:** 17.0.2 (Ruta manual en `/usr/lib/jvm/jdk-17.0.2`)
**IP del Servidor:** 192.168.0.8

---

## 1. Preparación del Sistema y Cuenta de Servicio
Se instalan las dependencias base de red y ejecución. *(Nota: Java 17 ya debe estar configurado globalmente)*.

```bash
sudo apt update
sudo apt install ssh pdsh wget tar rsync -y
```

**Aislamiento de Seguridad:** Se crea un grupo y un usuario de servicio exclusivo sin contraseña interactiva.

```bash
# Crear el grupo maestro para Big Data
sudo addgroup hadoop_group

# Crear el usuario de servicio (sin contraseña, con directorio Home)
sudo adduser --ingroup hadoop_group --disabled-password --gecos "" hadoop_user
```

## 2. Configuración de Seguridad SSH (Comunicación Interna)
Hadoop utiliza SSH para orquestar el arranque de sus nodos. Se generan llaves RSA exclusivas para la cuenta de servicio.

```bash
# Entrar al usuario de servicio
sudo su - hadoop_user

# Generar y autorizar llaves RSA
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
```

Se registra la huella de seguridad realizando una conexión manual (escribir `yes` y luego cerrar con `exit`).

```bash
ssh 192.168.0.8
exit
```

*(Nota: Al ejecutar exit, volverás a estar en la sesión del usuario hadoop_user. Sal nuevamente al administrador para el siguiente paso:)*
```bash
exit
```

## 3. Descarga de Binarios y Gestión de Propiedad
Desde la cuenta con privilegios de administrador, se descargan los binarios.

```bash
cd /tmp
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz
tar -xzvf hadoop-3.4.0.tar.gz
sudo mv hadoop-3.4.0 /usr/local/hadoop

# Entregar la propiedad absoluta al usuario y grupo de servicio
sudo chown -R hadoop_user:hadoop_group /usr/local/hadoop

# Asegurar los permisos del directorio Home de Hadoop (Logs y SSH)
sudo chown -R hadoop_user:hadoop_group /home/hadoop_user
sudo chmod 750 /home/hadoop_user
```

## 4. Variables de Entorno del Ecosistema
Se configuran las variables globales para la cuenta de servicio.

```bash
# Ingresar al usuario de Hadoop
sudo su - hadoop_user
nano ~/.bashrc
```

Añadir al final del archivo:

```bash
export JAVA_HOME=/usr/lib/jvm/jdk-17.0.2
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
```

Aplicar cambios en la sesión actual:
```bash
source ~/.bashrc
```

## 5. Configuración de Arquitectura (XML) y Parches de Compatibilidad
*Todos los comandos de esta sección deben ejecutarse bajo el usuario `hadoop_user`.*

**A. Creación de Directorios Físicos de Datos:**
```bash
mkdir -p /usr/local/hadoop/data/nameNode
mkdir -p /usr/local/hadoop/data/dataNode
```

**B. Archivos de Entorno (Parche Java 17):**
```bash
echo "export JAVA_HOME=/usr/lib/jvm/jdk-17.0.2" >> /usr/local/hadoop/etc/hadoop/hadoop-env.sh
echo "export PDSH_RCMD_TYPE=ssh" >> /usr/local/hadoop/etc/hadoop/hadoop-env.sh
echo 'export HADOOP_OPTS="$HADOOP_OPTS --add-opens java.base/java.lang=ALL-UNNAMED"' >> /usr/local/hadoop/etc/hadoop/hadoop-env.sh
echo 'export YARN_OPTS="$YARN_OPTS --add-opens java.base/java.lang=ALL-UNNAMED"' >> /usr/local/hadoop/etc/hadoop/yarn-env.sh
```

**C. core-site.xml (Cerebro del Clúster):**
```bash
cat << 'EOF' > /usr/local/hadoop/etc/hadoop/core-site.xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.0.8:9000</value>
    </property>
</configuration>
EOF
```

**D. hdfs-site.xml (Almacenamiento):**
```bash
cat << 'EOF' > /usr/local/hadoop/etc/hadoop/hdfs-site.xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/usr/local/hadoop/data/nameNode</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/usr/local/hadoop/data/dataNode</value>
    </property>
</configuration>
EOF
```

**E. mapred-site.xml (MapReduce):**
```bash
cat << 'EOF' > /usr/local/hadoop/etc/hadoop/mapred-site.xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
</configuration>
EOF
```

**F. yarn-site.xml (Gestión de Recursos):**
```bash
cat << 'EOF' > /usr/local/hadoop/etc/hadoop/yarn-site.xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>192.168.0.8</value>
    </property>
</configuration>
EOF
```

**G. Enrutamiento de Nodos:**
```bash
echo "192.168.0.8" > /usr/local/hadoop/etc/hadoop/workers
```

## 6. Inicialización de HDFS
Se formatea el NameNode con el usuario de servicio para construir el esquema base del sistema de archivos distribuido.

```bash
hdfs namenode -format
```
Una vez confirmado el formato exitoso, se retorna al usuario administrador del sistema:
```bash
exit
```

## 7. Automatización de Demonios (Systemd)
Para integrar Hadoop como un servicio resiliente, se crean unidades de control.

**Servicio HDFS (Sistema de Archivos)**
```bash
sudo nano /etc/systemd/system/hadoop-dfs.service
```
```ini
[Unit]
Description=Hadoop Distributed File System (HDFS)
After=network.target

[Service]
Type=forking
RemainAfterExit=yes
User=hadoop_user
Group=hadoop_group
Environment="HADOOP_HOME=/usr/local/hadoop"
Environment="JAVA_HOME=/usr/lib/jvm/jdk-17.0.2"
Environment="PDSH_RCMD_TYPE=ssh"
ExecStart=/usr/local/hadoop/sbin/start-dfs.sh
ExecStop=/usr/local/hadoop/sbin/stop-dfs.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**Servicio YARN (Gestor de Recursos)**
```bash
sudo nano /etc/systemd/system/hadoop-yarn.service
```
```ini
[Unit]
Description=Hadoop YARN Resource Manager
After=hadoop-dfs.service

[Service]
Type=forking
RemainAfterExit=yes
User=hadoop_user
Group=hadoop_group
Environment="HADOOP_HOME=/usr/local/hadoop"
Environment="JAVA_HOME=/usr/lib/jvm/jdk-17.0.2"
Environment="PDSH_RCMD_TYPE=ssh"
ExecStart=/usr/local/hadoop/sbin/start-yarn.sh
ExecStop=/usr/local/hadoop/sbin/stop-yarn.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## 8. Despliegue y Validación
Se habilitan e inician los servicios recién creados:

```bash
sudo systemctl daemon-reload
sudo systemctl enable hadoop-dfs.service
sudo systemctl enable hadoop-yarn.service
sudo systemctl start hadoop-dfs.service
sudo systemctl start hadoop-yarn.service
```

**Auditoría de Procesos Activos:**


```bash
sudo su - hadoop_user
```

### 1. Edita el archivo `.bashrc`
```bash
nano ~/.bashrc
```

### 2. Modifica la línea del PATH
Busca casi al final la línea que dice:
`export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin`

Y cámbiala para agregar el `$JAVA_HOME/bin` justo después del `$PATH:`, dejándola exactamente así:
```bash
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
```
*(Guarda con `Ctrl+O`, `Enter`, y sal con `Ctrl+X`)*.

### 3. Recarga la configuración y prueba
Para que la terminal lea el cambio inmediatamente, ejecuta:
```bash
source ~/.bashrc
```

 lanza el comando:
```bash
jps
```

La arquitectura se considera operativa si retorna los siguientes subprocesos:
* `NameNode`
* `DataNode`
* `SecondaryNameNode`
* `ResourceManager`
* `NodeManager`
* `Jps`


## 9. Interfaces Web y Seguridad de Acceso (Firewall)

Por diseño de seguridad, las interfaces de administración web de Apache Hadoop operan en puertos específicos que deben ser expuestos explícitamente a través del firewall del sistema operativo. En la arquitectura de Hadoop 3.x, los puertos estándar de monitoreo difieren de las versiones legacy, requiriendo una configuración de red precisa.

### 9.1. Puertos de Administración Requeridos
Para la auditoría y monitoreo del clúster Single-Node, se requieren los siguientes accesos:

* **NameNode (HDFS):** Puerto TCP `9870` (Sustituye al antiguo 50070). Permite visualizar la salud general del sistema de archivos distribuido, la capacidad de almacenamiento disponible y explorar los directorios de datos.
* **ResourceManager (YARN):** Puerto TCP `8088`. Interfaz central para monitorear la asignación de memoria, recursos de CPU y el estado de ejecución de los trabajos de procesamiento de datos.

### 9.2. Apertura de Puertos en UFW
Desde la cuenta con privilegios de administrador del sistema, se aplican las reglas de red para permitir el tráfico HTTP hacia los demonios de Hadoop, manteniendo bloqueado el resto del tráfico no autorizado.

```bash
# Permitir tráfico web hacia el NameNode (HDFS)
sudo ufw allow 9870/tcp

# Permitir tráfico web hacia el ResourceManager (YARN)
sudo ufw allow 8088/tcp

# Recargar las políticas de seguridad para aplicar los cambios
sudo ufw reload

# Verificar el estado actual de los puertos expuestos
sudo ufw status
```

### 9.3. Verificación de Acceso Web
Una vez aplicadas las reglas del firewall, abrir un navegador web desde una estación de trabajo autorizada en la red y acceder a las siguientes URLs, sustituyendo `<IP_DEL_SERVIDOR>` por la dirección correspondiente

* **Dashboard de HDFS (NameNode):** `http://<IP_DEL_SERVIDOR>:9870`
* **Dashboard de YARN (ResourceManager):** `http://<IP_DEL_SERVIDOR>:8088`





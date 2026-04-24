
# Instalación de Apache Atlas 2.4.0

A diferencia de otros componentes del clúster, Apache Atlas requiere la compilación de su código fuente mediante Apache Maven. Se documenta la configuración del entorno para garantizar la compatibilidad de compilación con Java 17.

## 1. Instalación del Motor de Compilación (Apache Maven)
**Usuario requerido:** Administrador (server con permisos sudo)

```bash
cd /tmp
wget https://archive.apache.org/dist/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
tar -xzf apache-maven-3.9.6-bin.tar.gz
sudo mv apache-maven-3.9.6 /usr/local/maven
sudo chown -R hadoop_user:hadoop_group /usr/local/maven
```

## 2. Descarga del Código Fuente de Atlas
**Usuario requerido:** Administrador (server)

```bash
cd /tmp
wget https://downloads.apache.org/atlas/2.4.0/apache-atlas-2.4.0-sources.tar.gz
wget https://downloads.apache.org/atlas/2.4.0/apache-atlas-2.4.0-sources.tar.gz.asc
wget https://downloads.apache.org/atlas/KEYS

# Verificación de integridad criptográfica
gpg --import KEYS
gpg --verify apache-atlas-2.4.0-sources.tar.gz.asc apache-atlas-2.4.0-sources.tar.gz

# Descompresión y asignación de permisos temporales para compilación
tar -xzf apache-atlas-2.4.0-sources.tar.gz
sudo chown -R hadoop_user:hadoop_group /tmp/apache-atlas-sources-2.4.0
```

## 3. Configuración de Variables de Entorno (Maven)
**Usuario requerido:** hadoop_user
Iniciamos sesión con el usuario de servicio y agregamos las rutas de Maven al entorno:

```bash
sudo su - hadoop_user
nano ~/.bashrc
```

Agregar al final del archivo:
```bash
# Apache Maven
export MAVEN_HOME=/usr/local/maven
export PATH=$PATH:$MAVEN_HOME/bin
```

Aplicar los cambios:
```bash
source ~/.bashrc
```

## 4. Compilación del Código Fuente (Build)
**Usuario requerido:** hadoop_user
Se inyectan opciones a la JVM para evadir las restricciones de módulos (`InaccessibleObjectException`) introducidas en Java 17, permitiendo a Maven compilar los componentes antiguos de Atlas.

```bash
cd /tmp/apache-atlas-sources-2.4.0

# Inyección de parámetros para Java 17 y memoria de compilación
export MAVEN_OPTS="-Xms2g -Xmx4g -XX:MaxMetaspaceSize=1g --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED"

# Compilación con bases embebidas (Este proceso demora entre 20 y 40 min)
mvn clean -DskipTests package -Pdist,embedded-hbase-solr
```

## 5. Ubicación del Binario Compilado y Permisos Finales
**Usuario requerido:** Administrador (server)
Una vez que Maven indique `BUILD SUCCESS`, se extrae el binario generado y se traslada a su ruta de producción. Salimos del usuario de Hadoop (`exit`).

```bash
cd /tmp/apache-atlas-sources-2.4.0/distro/target
sudo tar -xzf apache-atlas-2.4.0-server.tar.gz
sudo mv apache-atlas-2.4.0 /usr/local/atlas

# Asignación de propiedad final al usuario de servicio
sudo chown -R hadoop_user:hadoop_group /usr/local/atlas
```

## 6. Variables de Entorno (Atlas)
**Usuario requerido:** hadoop_user

```bash
sudo su - hadoop_user
nano ~/.bashrc
```

Agregar al final del archivo:
```bash
# Apache Atlas
export ATLAS_HOME=/usr/local/atlas
export PATH=$PATH:$ATLAS_HOME/bin
```

Aplicar cambios:
```bash
source ~/.bashrc
```

## 7. Orquestación del Servicio (Systemd)
**Usuario requerido:** Administrador (server)
Se encapsula el arranque del catálogo de gobernanza en un demonio de Linux. Salimos del usuario de Hadoop (`exit`).

```bash
sudo nano /etc/systemd/system/atlas.service
```

Contenido del archivo:
```ini
[Unit]
Description=Apache Atlas Metadata Governance
After=network.target

[Service]
Type=forking
User=hadoop_user
Group=hadoop_group
Environment=JAVA_HOME=/usr/lib/jvm/jdk-17.0.2
WorkingDirectory=/usr/local/atlas
ExecStart=/usr/local/atlas/bin/atlas_start.py
ExecStop=/usr/local/atlas/bin/atlas_stop.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Habilitar e iniciar el servicio:
```bash
sudo systemctl daemon-reload
sudo systemctl enable atlas.service
sudo systemctl start atlas.service
```

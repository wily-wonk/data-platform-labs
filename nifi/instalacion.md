# Guía de Instalación: Apache NiFi 1.28.1 en Debian 10

Esta guía detalla el despliegue de Apache NiFi utilizando OpenJDK 17 en un entorno Debian 10, implementando políticas de mínimo privilegio y gestión mediante SystemD.

## 1. Dependencias Base
Actualización del sistema e instalación de utilidades esenciales, incluyendo el gestor de firewall.

```bash
sudo apt update
sudo apt install wget unzip ufw -y
```

## 2. Instalación de Java 17 (Método Global)
Dado que los repositorios de Debian 10 no incluyen OpenJDK 17, se descargan los binarios oficiales y se configuran en el directorio del sistema.

```bash
# Ir al directorio temporal
cd /tmp

# Descargar OpenJDK 17.0.2
wget https://download.java.net/java/GA/jdk17.0.2/dfd4a8d0985749f896bed50d7138ee7f/8/GPL/openjdk-17.0.2_linux-x64_bin.tar.gz

# Descomprimir el archivo
tar -xzf openjdk-17.0.2_linux-x64_bin.tar.gz

# Crear directorio estándar y mover Java
sudo mkdir -p /usr/lib/jvm
sudo mv jdk-17.0.2 /usr/lib/jvm/

# Asignar propiedad a root por seguridad
sudo chown -R root:root /usr/lib/jvm/jdk-17.0.2

# Limpiar el instalador
rm openjdk-17.0.2_linux-x64_bin.tar.gz

# Verificar la instalación correcta
/usr/lib/jvm/jdk-17.0.2/bin/java -version
```

## 3. Creación del Usuario de Servicio (Mínimo Privilegio)
NiFi se ejecutará bajo un usuario dedicado sin acceso a la terminal.

```bash
# Crear grupo y usuario nifi
sudo groupadd nifi
sudo useradd -r -m -d /opt/nifi -s /bin/false -g nifi nifi

# Verificar la creación
id nifi
```

## 4. Descarga e Instalación de Apache NiFi
Despliegue de los binarios en el directorio `/opt`.

```bash
cd /tmp
wget https://downloads.apache.org/nifi/1.28.1/nifi-1.28.1-bin.zip
sudo unzip nifi-1.28.1-bin.zip -d /opt/
sudo mv /opt/nifi-1.28.1/* /opt/nifi/
sudo rmdir /opt/nifi-1.28.1
rm nifi-1.28.1-bin.zip
```

## 5. Configuración de Red, Memoria y PID

### 5.1. Configuración de Red
Se configura NiFi para aceptar conexiones seguras desde cualquier interfaz.

```bash
sudo nano /opt/nifi/conf/nifi.properties
```
*Modificar las siguientes propiedades en el archivo:*
```properties
# 1. Deja la interfaz en blanco y mantén el host en 0.0.0.0
nifi.web.https.host=0.0.0.0
nifi.web.https.port=8443
nifi.web.https.network.interface.default=

# 2. En la sección del proxy, pon tu IP pero CON EL PUERTO INCLUIDO
nifi.web.proxy.host=192.168.0.8:8443
```

### 5.2. Configuración de JVM y PID
Ajuste del *Heap* de memoria para el entorno (2GB) y definición del directorio PID (Crítico para SystemD).

```bash
# Crear directorio para el PID
sudo mkdir -p /opt/nifi/run

# Editar configuración de arranque
sudo nano /opt/nifi/conf/bootstrap.conf
```
*Localizar y modificar los parámetros de memoria:*
```properties
java.arg.2=-Xms1024m
java.arg.3=-Xmx2048m
```
*Agregar al final del archivo la ruta del PID (usando el índice 20 para evitar conflictos con argumentos existentes):*
```properties
# Directorio para archivo PID (CRÍTICO para SystemD)
java.arg.20=-Dorg.apache.nifi.bootstrap.config.pid.dir=/opt/nifi/run
```

## 6. Archivo de Servicio SystemD
Creación del demonio para gestionar NiFi como servicio del sistema operativo.

```bash
sudo nano /etc/systemd/system/nifi.service
```
*Insertar el siguiente contenido:*
```ini
[Unit]
Description=Apache NiFi Data Orchestration
After=network.target

[Service]
Type=forking
User=nifi
Group=nifi
Environment="JAVA_HOME=/usr/lib/jvm/jdk-17.0.2"
WorkingDirectory=/opt/nifi
ExecStart=/opt/nifi/bin/nifi.sh start
ExecStop=/opt/nifi/bin/nifi.sh stop
ExecReload=/opt/nifi/bin/nifi.sh restart
PIDFile=/opt/nifi/run/nifi.pid
Restart=on-failure
RestartSec=10
LimitNOFILE=50000
LimitNPROC=10000

[Install]
WantedBy=multi-user.target
```

## 7. Permisos Finales, Firewall y Arranque

```bash
# 1. Asignar propiedad recursiva al usuario nifi
sudo chown -R nifi:nifi /opt/nifi

# 2. Otorgar permisos de ejecución a los scripts
sudo find /opt/nifi/bin -name "*.sh" -exec chmod +x {} \;

# 3. Configurar Firewall (Abrir SSH y HTTPS de NiFi)
sudo ufw allow 8443/tcp
sudo ufw allow 22/tcp
sudo ufw enable

# 4. Habilitar e iniciar el servicio
sudo systemctl daemon-reload
sudo systemctl enable nifi
sudo systemctl start nifi

# 5. Verificar estado
sudo systemctl status nifi
```

## 8. Primer Acceso y Credenciales
En su primer arranque, NiFi autogenera un usuario y contraseña y crea los certificados TLS necesarios. Este proceso demora aproximadamente 2 minutos.

```bash
# Extraer credenciales generadas de los logs
sudo grep -E "Generated Username|Generated Password" /opt/nifi/logs/nifi-app.log
```

**Acceso Web:**
1. Navegar a `https://<IP_DEL_SERVIDOR>:8443/nifi`
2. Aceptar el certificado autofirmado generado por NiFi.
3. Ingresar utilizando las credenciales extraídas del log.


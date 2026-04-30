
# Guía Completa de Configuración e Instalación de Hooks (Apache Atlas)
**Archivo:** `instalar_hooks.md`
**Objetivo:** Habilitar el envío de metadatos y linaje desde las diferentes herramientas del clúster hacia Apache Atlas para el Gobierno de Datos.

---

## 1. Consideraciones de Arquitectura
* **Zookeeper:** No requiere instalación de Hook. Actúa como coordinador del clúster y no procesa datos ni genera linaje.
* **Kafka / Debezium:** Para este flujo, Atlas detectará los tópicos de Kafka automáticamente cuando NiFi interactúe con ellos y reporte el linaje. No instalaremos un hook directo en Kafka Connect.

---

## 2. Configuración del Hook en Apache NiFi

Dado que Apache NiFi no incluye el componente de Atlas por defecto, se debe descargar el archivo `.nar` y moverlo a la carpeta de librerías saltando las restricciones de permisos.

### Paso 2.1: Descargar e instalar el NAR vía `/tmp`
Ejecutar los siguientes comandos en la terminal del servidor:

```bash
# 1. Ir a la carpeta temporal para evitar errores de "Permission denied"
cd /tmp/

# 2. Descargar el archivo NAR oficial (Versión 1.28.1)
wget [https://repo1.maven.org/maven2/org/apache/nifi/nifi-atlas-nar/1.28.1/nifi-atlas-nar-1.28.1.nar](https://repo1.maven.org/maven2/org/apache/nifi/nifi-atlas-nar/1.28.1/nifi-atlas-nar-1.28.1.nar)

# 3. Mover el archivo a la carpeta lib de NiFi usando sudo
sudo mv nifi-atlas-nar-1.28.1.nar /opt/nifi/lib/

# 4. Asignar la propiedad al usuario nifi
sudo chown nifi:nifi /opt/nifi/lib/nifi-atlas-nar-1.28.1.nar

# 5. Reiniciar el servicio de NiFi
sudo systemctl restart nifi
```

### Paso 2.2: Configuración en la Interfaz Web de NiFi
1. Ir al menú global (esquina superior derecha) > **Controller Settings**.
2. En la pestaña **Reporting Tasks**, hacer clic en `+`.
3. Buscar y agregar `ReportLineageToAtlas`.
4. En sus propiedades, configurar:
   * **Atlas URLs:** `http://[IP_DEL_SERVIDOR_ATLAS]:21000`
   * **Atlas Username / Password:** Credenciales de Atlas.
5. Iniciar la tarea (botón Play).

---

## 3. Configuración del Hook en Apache Hive

Para evitar errores de `ClassNotFoundException` al momento de que Hive intente reportar a Atlas, es estrictamente necesario inyectar físicamente los archivos JAR de Atlas en las librerías de Hive.

### Paso 3.1: Copiar los JARs del Hook
```bash
# Copiar JARs del hook de Hive a la librería de Hive
sudo cp /usr/local/atlas/hook/hive/atlas-hive-plugin-impl/*.jar /usr/local/hive/lib/
sudo cp /usr/local/atlas/hook/hive/atlas-plugin-classloader/*.jar /usr/local/hive/lib/

# Ajustar permisos para el usuario de Hadoop
sudo chown hadoop_user:hadoop_user /usr/local/hive/lib/atlas-*.jar
```

### Paso 3.2: Modificar `hive-site.xml`
Editar el archivo de configuración:
```bash
sudo nano /usr/local/hive/conf/hive-site.xml
```
Agregar la siguiente propiedad. *(Nota: Si la propiedad ya existe y tiene otros hooks, separarlos por comas para no romper configuraciones previas).*
```xml
<property>
  <name>hive.exec.post.hooks</name>
  <value>org.apache.atlas.hive.hook.HiveHook</value>
</property>
```

### Paso 3.3: Reiniciar Servicios de Hive
```bash
sudo systemctl restart hive-server2.service
```

---

## 4. Configuración del Hook en Apache HBase

HBase utiliza un mecanismo llamado "Coprocessors" para inyectar lógica. Debemos copiar las librerías de Atlas y registrar el Coprocessor en el Master de HBase.

### Paso 4.1: Copiar los JARs del Hook
```bash
# Copiar JARs del hook de HBase a la librería de HBase
sudo cp /usr/local/atlas/hook/hbase/atlas-hbase-plugin-impl/*.jar /usr/local/hbase/lib/
sudo cp /usr/local/atlas/hook/hbase/atlas-plugin-classloader/*.jar /usr/local/hbase/lib/

# Ajustar permisos para el usuario de Hadoop
sudo chown hadoop_user:hadoop_user /usr/local/hbase/lib/atlas-*.jar
```

### Paso 4.2: Modificar `hbase-site.xml`
Editar el archivo de configuración:
```bash
sudo nano /usr/local/hbase/conf/hbase-site.xml
```
Agregar la propiedad para habilitar el coprocessor en el nodo Master:
```xml
<property>
  <name>hbase.coprocessor.master.classes</name>
  <value>org.apache.atlas.hbase.hook.HBaseAtlasCoprocessor</value>
</property>
```

### Paso 4.3: Reiniciar Servicios de HBase
*(El comando puede variar dependiendo de cómo se gestionen los servicios en el servidor, comúnmente usando los scripts bin)*:
```bash
sudo su - hadoop_user
/usr/local/hbase/bin/stop-hbase.sh
/usr/local/hbase/bin/start-hbase.sh
```

```

***

¡Listo! Con este documento tienes la base técnica sólida. ¿Quieres que vayamos revisando si NiFi levantó correctamente el componente en la interfaz web, o prefieres proceder con reiniciar los servicios de Hive y HBase primero?

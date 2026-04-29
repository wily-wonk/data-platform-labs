
# Atlas 2.4.0 + Solr 9 - Troubleshooting

> Problemas encontrados durante la instalación de Apache Atlas 2.4.0 con Solr 9 y Java 17/11 en Debian 10.

---

## 1. `ca-certificates-java` rompe la instalación de Java 11

**Síntoma:**
```
Exception in thread "main" java.lang.InternalError: Error loading java.security file
dpkg: error processing package ca-certificates-java (--configure)
```

**Causa:** El paquete `ca-certificates-java` de Debian 10 busca `/etc/ssl/certs/java/cacerts` y no existe.

**Solución:**
```bash
sudo mkdir -p /etc/ssl/certs/java
sudo touch /etc/ssl/certs/java/cacerts
sudo dpkg --force-depends --configure ca-certificates-java
sudo dpkg --force-depends --configure openjdk-11-jre-headless
sudo dpkg --force-depends --configure openjdk-11-jdk
```

---

## 2. Maven no puede descargar dependencias por error SSL

**Síntoma:**
```
NoSuchAlgorithmException: Error constructing implementation (algorithm: Default, provider: SunJSSE)
problem accessing trust store: Short read of DER length
```

**Causa:** El keystore de Java 11 está corrupto o vacío.

**Solución:**
```bash
sudo cp /usr/lib/jvm/jdk-17.0.2/lib/security/cacerts \
     /usr/lib/jvm/java-11-openjdk-amd64/lib/security/cacerts
```

---

## 3. Compilación falla en `atlas-dashboardv2`: `not found: make`

**Síntoma:**
```
gyp ERR! stack Error: not found: make
gyp ERR! not ok
Build failed with error code: 1
```

**Causa:** Falta `build-essential` (make, gcc, g++).

**Solución:**
```bash
sudo apt install -y build-essential
# Luego reanudar compilación con -rf :atlas-dashboardv2
```

---

## 4. Compilación falla en `atlas-webapp`: `enunciate` no compatible con Java 17

**Síntoma:**
```
Failed to execute goal com.webcohesion.enunciate:enunciate-maven-plugin:2.13.2:docs
class com.webcohesion.enunciate.Enunciate cannot access jdk.compiler
```

**Causa:** El plugin `enunciate` usa APIs internas de Java que están encapsuladas en Java 17.

**Solución:** Editar `atlas-webapp/pom.xml` y agregar `<skip>true</skip>`:
```xml
<plugin>
    <groupId>com.webcohesion.enunciate</groupId>
    <artifactId>enunciate-maven-plugin</artifactId>
    <configuration>
        <skip>true</skip>
    </configuration>
</plugin>
```

---

## 5. Atlas no arranca: `NoSuchFieldException: modifiers`

**Síntoma:**
```
java.lang.NoSuchFieldException: modifiers
at AtlasJanusGraphDatabase.addHBase2Support()
Caused by: java.lang.ExceptionInInitializerError
```

**Causa:** `AtlasJanusGraphDatabase` usa reflexión para acceder a `Field.modifiers`, campo eliminado en Java 17.

**Solución:** Ejecutar Atlas con **Java 11** en una burbuja aislada. En `atlas-env.sh`:
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

El resto del ecosistema (Kafka, HBase, Solr, NiFi) sigue en Java 17.

---

## 6. Error SLF4J: `log4j-over-slf4j.jar AND bound slf4j-log4j12.jar`

**Síntoma:**
```
SLF4J(E): Detected both log4j-over-slf4j.jar AND bound slf4j-log4j12.jar
Exception in thread "main" java.lang.ExceptionInInitializerError
```

**Causa:** El war de Atlas incluye JARs conflictivos de logging.

**Solución:** Después de instalar Atlas y extraer el war, renombrar los conflictivos:
```bash
cd /usr/local/atlas/server/webapp/atlas/WEB-INF/lib/
mv slf4j-reload4j-2.0.16.jar slf4j-reload4j-2.0.16.jar.bak
mv log4j-over-slf4j-2.0.16.jar log4j-over-slf4j-2.0.16.jar.bak
find /usr/local/atlas -name "atlas-testtools*.jar" -not -name "*.bak" -exec mv {} {}.bak \;
```

**IMPORTANTE:** `logback-classic` DEBE conservarse porque Atlas 2.4.0 lo necesita.

---

## 7. `503 Service Unavailable` después de `Apache Atlas Server started!!!`

**Síntoma:**
```
curl http://localhost:21000 → 503 Service Unavailable
Pero el log dice "Apache Atlas Server started!!!"
```

**Causa 1:** Conflicto SLF4J no resuelto.
**Causa 2:** `BeanUtil.context is null` → Spring no carga por error anterior.

**Solución:** Revisar `logs/atlas.*.err` y resolver el error raíz (generalmente SLF4J o Java 17).

---

## 8. Atlas se cuelga en `SetupSteps` y nunca crea tablas

**Síntoma:**
```
Running setup per configuration atlas.server.run.setup.on.start.
(se queda pegado aquí por siempre)
```

**Causa:** No puede conectarse a HBase/Solr/Kafka.

**Solución:**
- Verificar que HBase, Solr y Kafka estén corriendo
- En `atlas-application.properties` usar `hbase2` (no `hbase`)
- Asegurar ZooKeeper quorum correcto
- Crear tabla manualmente si es necesario:
  ```bash
  echo "create 'apache_atlas_janus', 'f'" | hbase shell -n
  ```

---


# Guía Completa de Configuración e Instalación de Hooks (Apache Atlas)
**Archivo:** `instalar_hooks.md`
**Objetivo:** Habilitar el envío de metadatos y linaje desde las diferentes herramientas del clúster hacia Apache Atlas para el Gobierno de Datos.

---

## 1. Consideraciones de Arquitectura
* **Zookeeper:** No requiere instalación de Hook. Actúa como coordinador del clúster y no procesa datos ni genera linaje.
* **Kafka / Debezium:** Para este flujo, Atlas detectará los tópicos de Kafka automáticamente cuando NiFi interactúe con ellos y reporte el linaje. No instalaremos un hook directo en Kafka Connect.

---

## 2. Configuración del Hook en Apache NiFi (ReportLineageToAtlas)

### Paso 2.1: Descargar e instalar el NAR

```bash
cd /tmp/
wget https://repo1.maven.org/maven2/org/apache/nifi/nifi-atlas-nar/1.28.1/nifi-atlas-nar-1.28.1.nar
sudo mv nifi-atlas-nar-1.28.1.nar /opt/nifi/lib/
sudo chown nifi:nifi /opt/nifi/lib/nifi-atlas-nar-1.28.1.nar
sudo systemctl restart nifi
```

### Paso 2.2: Configurar Atlas para NiFi

Agregar la propiedad de notificación síncrona en Atlas:

```bash
echo "atlas.notification.hook.asynchronous=false" | sudo tee -a /usr/local/atlas/conf/atlas-application.properties
sudo systemctl restart atlas
```

### Paso 2.3: Configurar el Reporting Task en NiFi

Ir a **Controller Settings** (menú ☰) → **Reporting Tasks** → **+** → buscar `ReportLineageToAtlas`

| Propiedad | Valor |
|-----------|-------|
| **Atlas URLs** | `http://localhost:21000` |
| **Atlas Configuration Directory** | `/usr/local/atlas/conf` |
| **Atlas Default Metadata Namespace** | `default` |
| **Create Atlas Configuration File** | `false` |
| **Lineage Strategy** | `Simple Path` |
| **NiFi URL for Atlas** | `https://gmlpsr014db048:8443/nifi` |
| **Atlas Authentication Method** | `Basic` |
| **Atlas Username** | `admin` |
| **Atlas Password** | `admin` |
| **Kafka Bootstrap Servers** | `ip-server:9092` |

### Paso 2.4: Verificar en Atlas

1. Ejecutar un flujo en NiFi (ej: `QueryDatabaseTable → PutHDFS`)
2. Esperar 1 minuto
3. Ir a Atlas → Search → buscar `nifi` o `fs_path`
¡Listo! Con este documento tienes la base técnica sólida. ¿Quieres que vayamos revisando si NiFi levantó correctamente el componente en la interfaz web, o prefieres proceder con reiniciar los servicios de Hive y HBase primero?

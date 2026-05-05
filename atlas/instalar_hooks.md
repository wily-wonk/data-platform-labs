
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

---

# GUÍA DE  CONFIGURACIÓN DE APACHE ATLAS 2.4.0 CON HIVE 4.0.0 Y HBASE 2.5.x

## Requisitos Previos

- Java 11 para Atlas / Java 17 para Hadoop, Hive y HBase
- Hadoop 3.4.0 funcionando
- HBase 2.5.x funcionando
- Hive 4.0.0 funcionando con Metastore (PostgreSQL)
- Solr 8.x para índices de Atlas
- Usuario `hadoop_user` con permisos adecuados
- Usuario con sudo (para instalación)

---

## FASE 1: Estabilización del Servidor - Eliminación de Hooks en Vivo

### ⚠️Problema
Los hooks en vivo de Atlas 2.4.0 inyectan librerías incompatibles (SLF4J, Guava, Jersey) que causan crashes en HBase y Hive con Java 17.

### 🔧 Solución

#### 1.1 Limpiar HBase

```bash
# Editar archivo de configuración de HBase
sudo nano /usr/local/hbase/conf/hbase-site.xml
```

**Eliminar** la propiedad:
```xml
<property>
  <name>hbase.coprocessor.master.classes</name>
  <value>org.apache.atlas.hbase.hook.HBaseAtlasCoprocessor</value>
</property>
```

**Reiniciar HBase:**
```bash
sudo systemctl restart hbase-master hbase-regionserver
# o
sudo /usr/local/hbase/bin/stop-hbase.sh && sudo /usr/local/hbase/bin/start-hbase.sh
```

**Verificar:**
```bash
echo "status" | /usr/local/hbase/bin/hbase shell -n
# Debe mostrar 1 active master y region servers
```

#### 1.2 Limpiar Hive

```bash
# Editar archivo de configuración de Hive
sudo nano /usr/local/hive/conf/hive-site.xml
```

**Eliminar** la propiedad:
```xml
<property>
  <name>hive.exec.post.hooks</name>
  <value>org.apache.atlas.hive.hook.HiveHook</value>
</property>
```

**Reiniciar HiveServer2:**
```bash
sudo systemctl restart hive-server2
# o
sudo pkill -f HiveServer2
sudo /usr/local/hive/bin/hive --service hiveserver2 &
```

**Verificar:**
```bash
beeline -u "jdbc:hive2://localhost:10000" -n hadoop_user --silent=true --outputformat=tsv2 -e "show databases;"
# Debe mostrar las bases de datos sin errores
```

---

## FASE 2: Conexión de Atlas con HBase (Almacenamiento)

### ⚠️ Problema
Atlas 2.4.0 no puede crear sus propias tablas en HBase 2.5.x por incompatibilidad de API.

### 🔧 Solución

#### 2.1 Crear tablas manualmente en HBase

```bash
# Como hadoop_user o usuario con acceso a HBase shell
echo "create 'apache_atlas_janus', 's'" | /usr/local/hbase/bin/hbase shell -n
echo "create 'apache_atlas_entity_audit', 'dt'" | /usr/local/hbase/bin/hbase shell -n
```

**Verificar tablas:**
```bash
echo "list" | /usr/local/hbase/bin/hbase shell -n
# Debe mostrar:
# TABLE
# apache_atlas_entity_audit
# apache_atlas_janus
```

#### 2.2 Configurar Atlas para conexión con HBase y Solr

```bash
# Editar configuración de Atlas
sudo nano /usr/local/atlas/conf/atlas-application.properties
```

**Asegurar que contiene:**
```properties
# HBase configuration
atlas.graph.storage.hostname=localhost
atlas.graph.storage.hbase.table=apache_atlas_janus
atlas.graph.storage.hbase.regions-per-server=1

# Solr configuration
atlas.graph.index.search.backend=solr
atlas.graph.index.search.solr.mode=cloud
atlas.graph.index.search.solr.zookeeper-url=localhost:2181
atlas.graph.index.search.solr.config-name=atlas_index

# Audit configuration
atlas.audit.hbase.tablename=apache_atlas_entity_audit
```

#### 2.3 Iniciar Atlas

```bash
sudo systemctl start atlas
# o
sudo /usr/local/atlas/bin/atlas_start.py
```

**Verificar inicio exitoso:**
```bash
tail -f /var/log/atlas/application.log
# Debe mostrar: "Apache Atlas Server started!!!"
```

**Verificar UI:**
```
http://<IP_DEL_SERVIDOR>:21000
# Login: admin / admin
```

---

## FASE 3: Preparación del Puente Hive-Atlas (Importación Asíncrona)

### ⚠️ Problema
El script nativo `import-hive.sh` de Atlas 2.4.0 tiene incompatibilidades con:
- Jersey/Jackson en Java 11
- Múltiples versiones de SLF4J
- Serialización JSON de entidades Atlas

### 🔧 Solución: Script alternativo vía API REST (Nivel Enterprise)

#### 3.1 Crear el script de importación mejorado

```bash
sudo tee /usr/local/bin/import-hive-rest.sh > /dev/null << 'SCRIPT'
#!/bin/bash
###############################################################################
# Script de Importación de Metadatos Hive -> Atlas vía REST API
# Versión: 2.0 Enterprise
# Compatible con Atlas 2.4.0 + Hive 4.0.0 + Java 11/17
# 
# Características:
# - Silencia completamente la salida de Beeline (datos puros)
# - Manejo robusto de errores
# - Logging estructurado
# - Soporte para múltiples bases de datos y tablas
# - Verificación de conectividad previa
#
# Uso: /usr/local/bin/import-hive-rest.sh
###############################################################################

set -o pipefail  # Capturar errores en tuberías

# ===== CONFIGURACIÓN =====
ATLAS_URL="${ATLAS_URL:-http://localhost:21000}"
ATLAS_USER="${ATLAS_USER:-admin}"
ATLAS_PASS="${ATLAS_PASS:-admin}"
BEELINE_URL="${BEELINE_URL:-jdbc:hive2://localhost:10000}"
BEELINE_USER="${BEELINE_USER:-hadoop_user}"
LOG_FILE="${LOG_FILE:-/var/log/atlas/import-hive-rest.log}"

# Opciones de Beeline para salida limpia
BEELINE_OPTS="--silent=true --outputformat=tsv2 --showHeader=false --incremental=false"

# Crear directorio de logs si no existe
mkdir -p "$(dirname "$LOG_FILE")" 2>/dev/null

# ===== FUNCIÓN DE LOGGING =====
log() {
    local level="$1"
    shift
    local message="$@"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[${timestamp}] [${level}] ${message}" | tee -a "$LOG_FILE"
}

log_info() { log "INFO" "$@"; }
log_error() { log "ERROR" "$@"; }
log_success() { log "SUCCESS" "$@"; }
log_warn() { log "WARN" "$@"; }

# ===== FUNCIÓN: VERIFICAR CONECTIVIDAD =====
check_connectivity() {
    log_info "Verificando conectividad con HiveServer2..."
    
    local hive_test=$(beeline -u "${BEELINE_URL}" -n ${BEELINE_USER} \
        ${BEELINE_OPTS} \
        -e "select 1;" 2>/dev/null | tr -d '[:space:]')
    
    if [ "$hive_test" != "1" ]; then
        log_error "No se puede conectar a HiveServer2 en ${BEELINE_URL}"
        log_error "Verifica que HiveServer2 esté corriendo"
        return 1
    fi
    log_success "Conexión a HiveServer2 exitosa"
    
    log_info "Verificando conectividad con Atlas..."
    local atlas_test=$(curl -s -o /dev/null -w "%{http_code}" \
        -u ${ATLAS_USER}:${ATLAS_PASS} \
        "${ATLAS_URL}/api/atlas/v2/types")
    
    if [ "$atlas_test" != "200" ]; then
        log_error "No se puede conectar a Atlas en ${ATLAS_URL} (HTTP ${atlas_test})"
        log_error "Verifica que Atlas esté corriendo"
        return 1
    fi
    log_success "Conexión a Atlas exitosa"
    
    return 0
}

# ===== FUNCIÓN: CREAR ENTIDAD EN ATLAS =====
create_atlas_entity() {
    local entity_json="$1"
    local entity_type="$2"
    local entity_name="$3"
    
    local response=$(curl -s -w "\n%{http_code}" \
        -u ${ATLAS_USER}:${ATLAS_PASS} \
        -X POST \
        -H "Content-Type: application/json" \
        -d "$entity_json" \
        "${ATLAS_URL}/api/atlas/v2/entity")
    
    local http_code=$(echo "$response" | tail -1)
    local body=$(echo "$response" | sed '$d')
    
    case $http_code in
        200|201)
            # Extraer GUID de la respuesta
            local guid=$(echo "$body" | grep -o '"guid":"[^"]*"' | head -1 | cut -d'"' -f4)
            local operation=$(echo "$body" | grep -o '"CREATE"' > /dev/null && echo "CREADO" || echo "ACTUALIZADO")
            log_success "${entity_type} '${entity_name}' ${operation} (GUID: ${guid})"
            return 0
            ;;
        401)
            log_error "Autenticación fallida para Atlas. Verifica credenciales."
            return 1
            ;;
        404)
            log_error "Endpoint de Atlas no encontrado. Verifica versión de Atlas."
            return 1
            ;;
        500)
            log_error "Error interno de Atlas al crear ${entity_type} '${entity_name}'"
            log_error "Respuesta: ${body}"
            return 1
            ;;
        *)
            log_error "Error HTTP ${http_code} al crear ${entity_type} '${entity_name}'"
            return 1
            ;;
    esac
}

# ===== FUNCIÓN: OBTENER BASES DE DATOS =====
get_databases() {
    beeline -u "${BEELINE_URL}" -n ${BEELINE_USER} \
        ${BEELINE_OPTS} \
        -e "show databases;" 2>/dev/null \
        | grep -v "^$" \
        | grep -v "database_name" \
        | tr -d '[:space:]'
}

# ===== FUNCIÓN: OBTENER INFORMACIÓN DE BD =====
get_database_info() {
    local db="$1"
    beeline -u "${BEELINE_URL}" -n ${BEELINE_USER} \
        ${BEELINE_OPTS} \
        -e "describe database ${db};" 2>/dev/null
}

# ===== FUNCIÓN: OBTENER TABLAS =====
get_tables() {
    local db="$1"
    beeline -u "${BEELINE_URL}" -n ${BEELINE_USER} \
        ${BEELINE_OPTS} \
        -e "use ${db}; show tables;" 2>/dev/null \
        | grep -v "^$" \
        | grep -v "tab_name" \
        | tr -d '[:space:]'
}

# ===== INICIO DEL SCRIPT =====
log_info "======================================================================="
log_info "  INICIANDO IMPORTACIÓN DE METADATOS HIVE -> ATLAS"
log_info "  Fecha: $(date)"
log_info "  Atlas: ${ATLAS_URL}"
log_info "  Hive:  ${BEELINE_URL}"
log_info "======================================================================="

# Verificar conectividad
check_connectivity
if [ $? -ne 0 ]; then
    log_error "Abortando importación por fallos de conectividad"
    exit 1
fi

# ===== PASO 1: OBTENER BASES DE DATOS =====
log_info "[PASO 1/3] Obteniendo bases de datos de Hive..."

DBS=$(get_databases)

if [ -z "$DBS" ]; then
    log_error "No se encontraron bases de datos en Hive"
    exit 1
fi

DB_COUNT=$(echo "$DBS" | wc -l)
log_info "  Se encontraron ${DB_COUNT} base(s) de datos"

# ===== PASO 2: PROCESAR BASES DE DATOS Y TABLAS =====
log_info "[PASO 2/3] Procesando bases de datos y tablas..."
echo ""

TOTAL_TABLES=0
DB_IMPORTED=0
TABLE_IMPORTED=0
DB_ERRORS=0
TABLE_ERRORS=0

for DB in $DBS; do
    log_info "─────────────────────────────────────────"
    log_info "  Procesando base de datos: '${DB}'"
    
    # Obtener información de la base de datos
    DB_INFO=$(get_database_info "$DB")
    
    DB_LOCATION=$(echo "$DB_INFO" | grep -i "location:" | awk -F'\t' '{print $2}' | tr -d '[:space:]')
    DB_OWNER=$(echo "$DB_INFO" | grep -i "owner:" | awk -F'\t' '{print $2}' | tr -d '[:space:]')
    
    [ -z "$DB_LOCATION" ] && DB_LOCATION="hdfs://localhost:9000/user/hive/warehouse/${DB}.db"
    [ -z "$DB_OWNER" ] && DB_OWNER="${BEELINE_USER}"
    
    # Crear entidad de base de datos en Atlas
    DB_ENTITY=$(cat <<EOF
{
  "entity": {
    "typeName": "hive_db",
    "attributes": {
      "qualifiedName": "${DB}@primary",
      "name": "${DB}",
      "clusterName": "primary",
      "owner": "${DB_OWNER}",
      "ownerType": "USER",
      "description": "Database ${DB} (imported from Hive)",
      "location": "${DB_LOCATION}",
      "parameters": {}
    }
  }
}
EOF
)
    
    create_atlas_entity "$DB_ENTITY" "hive_db" "$DB"
    if [ $? -eq 0 ]; then
        DB_IMPORTED=$((DB_IMPORTED + 1))
    else
        DB_ERRORS=$((DB_ERRORS + 1))
    fi
    
    # Obtener tablas de la base de datos
    TABLES=$(get_tables "$DB")
    
    if [ -z "$TABLES" ]; then
        log_info "    No se encontraron tablas en '${DB}'"
        continue
    fi
    
    TABLE_COUNT=$(echo "$TABLES" | wc -l)
    log_info "    Se encontraron ${TABLE_COUNT} tabla(s)"
    
    # Procesar cada tabla
    for TABLE in $TABLES; do
        log_info "    Creando tabla: '${DB}.${TABLE}'"
        
        TABLE_ENTITY=$(cat <<EOF
{
  "entity": {
    "typeName": "hive_table",
    "attributes": {
      "qualifiedName": "${DB}.${TABLE}@primary",
      "name": "${TABLE}",
      "db": {
        "typeName": "hive_db",
        "uniqueAttributes": {
          "qualifiedName": "${DB}@primary"
        }
      },
      "tableType": "MANAGED_TABLE",
      "owner": "${DB_OWNER}",
      "description": "Table ${DB}.${TABLE} (imported from Hive)"
    }
  }
}
EOF
)
        
        create_atlas_entity "$TABLE_ENTITY" "hive_table" "${DB}.${TABLE}"
        if [ $? -eq 0 ]; then
            TABLE_IMPORTED=$((TABLE_IMPORTED + 1))
        else
            TABLE_ERRORS=$((TABLE_ERRORS + 1))
        fi
        TOTAL_TABLES=$((TOTAL_TABLES + 1))
    done
    
    echo ""
done

# ===== PASO 3: RESUMEN =====
log_info "[PASO 3/3] RESUMEN DE IMPORTACIÓN"
log_info "======================================================================="
log_info "  Bases de datos encontradas: ${DB_COUNT}"
log_info "  Bases de datos importadas:  ${DB_IMPORTED}"
log_info "  Bases de datos con error:   ${DB_ERRORS}"
log_info "  Tablas encontradas:         ${TOTAL_TABLES}"
log_info "  Tablas importadas:          ${TABLE_IMPORTED}"
log_info "  Tablas con error:           ${TABLE_ERRORS}"
log_info ""
log_info "  Verificar en UI:  ${ATLAS_URL}"
log_info "  Verificar con curl:"
log_info "    curl -u ${ATLAS_USER}:${ATLAS_PASS} ${ATLAS_URL}/api/atlas/v2/search/basic?typeName=hive_db"
log_info "    curl -u ${ATLAS_USER}:${ATLAS_PASS} ${ATLAS_URL}/api/atlas/v2/search/basic?typeName=hive_table"
log_info "======================================================================="

# Código de salida
if [ $DB_ERRORS -gt 0 ] || [ $TABLE_ERRORS -gt 0 ]; then
    exit 1
else
    exit 0
fi
SCRIPT

sudo chmod +x /usr/local/bin/import-hive-rest.sh
```

#### 3.2 Probar la importación mejorada

```bash
# Como hadoop_user
/usr/local/bin/import-hive-rest.sh
```

**Salida esperada:**
```
[2026-05-04 23:00:00] [INFO] =======================================================================
[2026-05-04 23:00:00] [INFO]   INICIANDO IMPORTACIÓN DE METADATOS HIVE -> ATLAS
[2026-05-04 23:00:00] [INFO]   Fecha: Mon May  4 23:00:00 -04 2026
[2026-05-04 23:00:00] [INFO]   Atlas: http://localhost:21000
[2026-05-04 23:00:00] [INFO]   Hive:  jdbc:hive2://localhost:10000
[2026-05-04 23:00:00] [INFO] =======================================================================
[2026-05-04 23:00:01] [SUCCESS] Conexión a HiveServer2 exitosa
[2026-05-04 23:00:01] [SUCCESS] Conexión a Atlas exitosa
[2026-05-04 23:00:01] [INFO] [PASO 1/3] Obteniendo bases de datos de Hive...
[2026-05-04 23:00:02] [INFO]   Se encontraron 1 base(s) de datos
[2026-05-04 23:00:02] [INFO] [PASO 2/3] Procesando bases de datos y tablas...
[2026-05-04 23:00:02] [INFO] ─────────────────────────────────────────
[2026-05-04 23:00:02] [INFO]   Procesando base de datos: 'default'
[2026-05-04 23:00:02] [SUCCESS] hive_db 'default' CREADO (GUID: 3ffa3b99-74f7-41ee-811a-702a08722518)
[2026-05-04 23:00:02] [INFO]     Se encontraron 1 tabla(s)
[2026-05-04 23:00:02] [INFO]     Creando tabla: 'default.test_atlas'
[2026-05-04 23:00:02] [SUCCESS] hive_table 'default.test_atlas' CREADO (GUID: 7c8b2a45-1234-5678-9abc-def012345678)
[2026-05-04 23:00:02] [INFO] [PASO 3/3] RESUMEN DE IMPORTACIÓN
[2026-05-04 23:00:02] [INFO] =======================================================================
[2026-05-04 23:00:02] [INFO]   Bases de datos encontradas: 1
[2026-05-04 23:00:02] [INFO]   Bases de datos importadas:  1
[2026-05-04 23:00:02] [INFO]   Bases de datos con error:   0
[2026-05-04 23:00:02] [INFO]   Tablas encontradas:         1
[2026-05-04 23:00:02] [INFO]   Tablas importadas:          1
[2026-05-04 23:00:02] [INFO]   Tablas con error:           0
[2026-05-04 23:00:02] [INFO] =======================================================================
```

---

## FASE 4: Automatización (Opcional)

### 4.1 Programar importación periódica

```bash
# Como hadoop_user
crontab -e
```

Añadir:
```cron
# Importar metadata de Hive a Atlas cada hora
0 * * * * /usr/local/bin/import-hive-rest.sh >> /var/log/atlas/import-hive-cron.log 2>&1

# O cada 30 minutos
*/30 * * * * /usr/local/bin/import-hive-rest.sh >> /var/log/atlas/import-hive-cron.log 2>&1
```

### 4.2 Monitoreo con logrotate

```bash
sudo tee /etc/logrotate.d/atlas-import > /dev/null << 'EOF'
/var/log/atlas/import-hive-rest.log
/var/log/atlas/import-hive-cron.log
{
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    maxsize 100M
}
EOF
```

---

## FASE 5: Resolución de Problemas Comunes

### 5.1 Si Atlas no inicia

```bash
# Verificar logs
tail -100 /var/log/atlas/application.log

# Problemas comunes:
# - HBase no está corriendo -> sudo systemctl start hbase-master hbase-regionserver
# - Solr no está corriendo -> sudo systemctl start solr
# - Tablas no existen -> Crear manualmente (FASE 2)
# - Problemas de permisos -> sudo chown -R hadoop_user:hadoop_group /usr/local/atlas/
```

### 5.2 Si la importación REST falla

```bash
# Verificar conectividad
beeline -u "jdbc:hive2://localhost:10000" -n hadoop_user --silent=true --outputformat=tsv2 -e "select 1;"
curl -u admin:admin http://localhost:21000/api/atlas/v2/types

# Verificar tipos de entidad en Atlas
curl -u admin:admin http://localhost:21000/api/atlas/v2/types/typedefs?type=hive_db
curl -u admin:admin http://localhost:21000/api/atlas/v2/types/typedefs?type=hive_table

# Ver log detallado
tail -f /var/log/atlas/import-hive-rest.log
```

### 5.3 Si necesitas reinstalar los hooks para otra funcionalidad

```bash
# Extraer binarios del hook
sudo tar -xzf /tmp/apache-atlas-sources-2.4.0/distro/target/apache-atlas-2.4.0-hive-hook.tar.gz \
    -C /usr/local/atlas/ --strip-components=1

# Ajustar permisos
sudo chown -R hadoop_user:hadoop_group /usr/local/atlas/hook/
sudo chown -R hadoop_user:hadoop_group /usr/local/atlas/hook-bin/
```

---

## 📊 Diagrama de Arquitectura Final

```
┌─────────────────────────────────────────────────────────────┐
│                     SERVIDOR ATLAS/HIVE                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐     REST API      ┌──────────────────────┐   │
│  │  Script   │───────POST──────▶│    Apache Atlas       │   │
│  │ import-   │  (JSON/HTTP)     │    (Puerto 21000)     │   │
│  │ hive-rest │                  └──────────┬───────────┘   │
│  └─────┬─────┘                             │               │
│        │ Beeline                           │               │
│        │ (JDBC TSV2)                ┌──────▼───────────┐   │
│        │                            │   HBase + Solr    │   │
│  ┌─────▼─────┐                      │   (Almacenamiento)│   │
│  │  Hive     │                      └──────────────────┘   │
│  │  Server2  │                                             │
│  │  (10000)  │                                             │
│  └─────┬─────┘                                             │
│        │                                                   │
│  ┌─────▼─────┐                                             │
│  │ Metastore │                                             │
│  │(PostgreSQL│                                             │
│  └───────────┘                                             │
│                                                             │
│  Logs: /var/log/atlas/import-hive-rest.log                 │
└─────────────────────────────────────────────────────────────┘
```

---

## ✅ Checklist de Verificación Final

```bash
# 1. HBase funcionando
echo "status" | /usr/local/hbase/bin/hbase shell -n

# 2. Hive funcionando (salida limpia)
beeline -u "jdbc:hive2://localhost:10000" -n hadoop_user --silent=true --outputformat=tsv2 -e "show databases;"

# 3. Atlas funcionando
curl -u admin:admin http://localhost:21000/api/atlas/v2/types

# 4. Importación funcionando
/usr/local/bin/import-hive-rest.sh

# 5. Ver entidades en Atlas
curl -u admin:admin "http://localhost:21000/api/atlas/v2/search/basic?typeName=hive_db"
curl -u admin:admin "http://localhost:21000/api/atlas/v2/search/basic?typeName=hive_table"

# 6. Ver log de importación
cat /var/log/atlas/import-hive-rest.log
```

---

## 📝 Notas Importantes

1. **NUNCA volver a agregar los hooks en vivo** a Hive o HBase. Usar solo importación asíncrona vía REST.
2. **Si se actualiza Java**, mantener Java 11 para Atlas y Java 17 para el resto.
3. **Backup de configuraciones** antes de cualquier cambio:
   ```bash
   sudo mkdir -p /backup/atlas-$(date +%Y%m%d)
   sudo cp /usr/local/atlas/conf/atlas-application.properties /backup/atlas-$(date +%Y%m%d)/
   sudo cp /usr/local/hive/conf/hive-site.xml /backup/atlas-$(date +%Y%m%d)/
   sudo cp /usr/local/hbase/conf/hbase-site.xml /backup/atlas-$(date +%Y%m%d)/
   ```
4. **Monitoreo**: 
   - Log de importación: `/var/log/atlas/import-hive-rest.log`
   - Log de Atlas: `/var/log/atlas/application.log`
   - Rotación de logs configurada en `/etc/logrotate.d/atlas-import`
5. **El script usa TSV2** (Tab-Separated Values) para garantizar datos limpios sin formato.

---



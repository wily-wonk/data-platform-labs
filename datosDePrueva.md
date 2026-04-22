
#  **GUÍA COMPLETA: DESPLIEGUE DE 3 GESTORES DE BASES DE DATOS CON DATOS DE PRUEBA**

##  **Objetivo**
Desplegar PostgreSQL 10, MySQL 5.7 y SQL Server 2017 en Docker con datos de prueba para ejercicios de ingesta con Apache NiFi.

---

##  **Requisitos Previos**
- Docker Desktop instalado y corriendo
- PowerShell (Windows) o Terminal (Linux)
- Conexión a internet para descargar imágenes

---

##  **PASO 1: CREAR DOCKER-COMPOSE CON LOS 3 GESTORES**

```powershell
# Crear carpeta del proyecto
mkdir C:\ejercicio-multi-bd
cd C:\ejercicio-multi-bd

# Crear docker-compose.yml
@"
services:
  postgres:
    image: postgres:10
    container_name: postgres_gamlp
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: dd8)JBY)/2026
      POSTGRES_DB: negocios_db
    ports:
      - "5433:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./csv_data:/tmp/csv_data

  mysql:
    image: mysql:5.7
    container_name: mysql_gamlp
    environment:
      MYSQL_ROOT_PASSWORD: dd8)JBY)/2026
      MYSQL_DATABASE: negocios_db
      MYSQL_USER: admin
      MYSQL_PASSWORD: dd8)JBY)/2026
    ports:
      - "3307:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./csv_data:/tmp/csv_data

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2017-latest
    container_name: sqlserver_gamlp
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "dd8)JBY)/2026"
      MSSQL_PID: Developer
    ports:
      - "1434:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql
      - ./csv_data:/tmp/csv_data

volumes:
  postgres_data:
  mysql_data:
  sqlserver_data:
"@ | Out-File -FilePath docker-compose.yml -Encoding UTF8

# Crear carpeta para CSVs
mkdir csv_data

# Levantar contenedores
docker-compose up -d

# Esperar inicialización (SQL Server tarda más)
Write-Host "Esperando 45 segundos..." -ForegroundColor Yellow
Start-Sleep -Seconds 45

# Verificar estado
docker ps
```

---

##  **PASO 2: GENERAR DATOS DE PRUEBA (CSV)**

Usar el generador online: https://calcbe.com/es/generador-csv

### **2.1 Proveedores (PostgreSQL) - 1000 registros**
```json
{
  "columns": [
    {"name": "id", "type": "sequential", "params": {"start": 1, "step": 1}},
    {"name": "nombre", "type": "company_name", "params": {}},
    {"name": "contacto", "type": "full_name", "params": {}},
    {"name": "email", "type": "email", "params": {"domain": "proveedor.com"}},
    {"name": "telefono", "type": "phone", "params": {}},
    {"name": "saldo", "type": "float", "params": {"min": 100, "max": 10000}},
    {"name": "fecha_registro", "type": "datetime", "params": {"start": "2023-01-01", "end": "2024-12-31", "format": "YYYY-MM-DD"}}
  ],
  "rows": 1000,
  "delimiter": ",",
  "includeHeader": true
}
```
Guardar como `postgresdatos.csv`

### **2.2 Clientes (MySQL) - 1000 registros**
```json
{
  "columns": [
    {"name": "cliente_id", "type": "sequential", "params": {"start": 1000, "step": 1}},
    {"name": "nombre", "type": "full_name", "params": {}},
    {"name": "email", "type": "email", "params": {"domain": "cliente.com"}},
    {"name": "ciudad", "type": "city", "params": {}},
    {"name": "pais", "type": "country", "params": {}},
    {"name": "total_compras", "type": "float", "params": {"min": 0, "max": 50000}},
    {"name": "ultima_compra", "type": "datetime", "params": {"start": "2023-01-01", "end": "2024-12-31", "format": "YYYY-MM-DD"}}
  ],
  "rows": 1000,
  "delimiter": ",",
  "includeHeader": true
}
```
Guardar como `mysqldatos.csv`

### **2.3 Productos (SQL Server) - 1000 registros**
```json
{
  "columns": [
    {"name": "producto_id", "type": "uuid", "params": {}},
    {"name": "nombre", "type": "company_name", "params": {}},
    {"name": "categoria", "type": "city", "params": {}},
    {"name": "precio", "type": "float", "params": {"min": 10, "max": 5000}},
    {"name": "stock", "type": "integer", "params": {"min": 0, "max": 500}},
    {"name": "proveedor_id", "type": "integer", "params": {"min": 1, "max": 100}},
    {"name": "fecha_creacion", "type": "datetime", "params": {"start": "2022-01-01", "end": "2024-12-31", "format": "YYYY-MM-DD"}}
  ],
  "rows": 1000,
  "delimiter": ",",
  "includeHeader": true
}
```
Guardar como `sqlserverdatos.csv`

---

## **PASO 3: COPIAR CSVs A LA CARPETA DEL PROYECTO**

```powershell
# Mover archivos descargados
Move-Item "$env:USERPROFILE\Downloads\postgresdatos.csv" -Destination ".\" -Force
Move-Item "$env:USERPROFILE\Downloads\mysqldatos.csv" -Destination ".\" -Force
Move-Item "$env:USERPROFILE\Downloads\sqlserverdatos.csv" -Destination ".\" -Force

# Copiar a carpeta csv_data
Copy-Item ".\postgresdatos.csv" -Destination ".\csv_data\proveedores.csv"
Copy-Item ".\mysqldatos.csv" -Destination ".\csv_data\clientes.csv"
Copy-Item ".\sqlserverdatos.csv" -Destination ".\csv_data\productos.csv"
```

---

## **PASO 4: CREAR TABLAS Y CARGAR DATOS**

### **4.1 PostgreSQL - Tabla proveedores**
```powershell
docker exec -it postgres_gamlp psql -U admin -d negocios_db -c "
CREATE TABLE IF NOT EXISTS proveedores (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100),
    contacto VARCHAR(100),
    email VARCHAR(100),
    telefono VARCHAR(50),
    saldo FLOAT,
    fecha_registro DATE
);"

docker cp .\csv_data\proveedores.csv postgres_gamlp:/tmp/
docker exec -it postgres_gamlp psql -U admin -d negocios_db -c "
TRUNCATE proveedores;
COPY proveedores FROM '/tmp/proveedores.csv' DELIMITER ',' CSV HEADER;"
```

### **4.2 SQL Server - Tabla productos**
```powershell
docker exec -it sqlserver_gamlp /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "dd8)JBY)/2026" -Q "
DROP TABLE IF EXISTS productos;
CREATE TABLE productos (
    producto_id UNIQUEIDENTIFIER,
    nombre NVARCHAR(200),
    categoria NVARCHAR(100),
    precio FLOAT,
    stock INT,
    proveedor_id INT,
    fecha_creacion DATETIME
);"

docker cp .\csv_data\productos.csv sqlserver_gamlp:/tmp/
docker exec -it sqlserver_gamlp /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "dd8)JBY)/2026" -Q "
BULK INSERT productos
FROM '/tmp/productos.csv'
WITH (
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    FIRSTROW = 2,
    TABLOCK
);"
```

### **4.3 MySQL - Tabla clientes (GENERACIÓN DIRECTA)**
> **Nota:** MySQL tuvo problemas con LOAD DATA, por lo que se generan los datos directamente con un procedimiento almacenado.

```powershell
docker exec -it mysql_gamlp mysql -u root -p'dd8)JBY)/2026' -e "
USE negocios_db;
DROP TABLE IF EXISTS clientes;
CREATE TABLE clientes (
    cliente_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    email VARCHAR(100),
    ciudad VARCHAR(50),
    pais VARCHAR(50),
    total_compras DECIMAL(10,2),
    ultima_compra DATE
);"

# Generar 1000 registros con procedimiento almacenado
docker exec -it mysql_gamlp mysql -u root -p'dd8)JBY)/2026' -e "
USE negocios_db;
DELIMITER //
CREATE PROCEDURE generate_clients(IN num_rows INT)
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= num_rows DO
        INSERT INTO clientes VALUES (
            i + 1000,
            CONCAT(ELT(1 + FLOOR(RAND() * 10), 'Juan','Ana','Carlos','María','Pedro','Laura','Javier','Carmen','Antonio','Isabel'),
                   ' ',
                   ELT(1 + FLOOR(RAND() * 10), 'García','Rodríguez','López','Martínez','Sánchez','Pérez','Gómez','Fernández','Ruiz','Díaz')),
            CONCAT('cliente', i, '@email.com'),
            ELT(1 + FLOOR(RAND() * 5), 'Madrid','Barcelona','Valencia','Sevilla','Bilbao'),
            'España',
            ROUND(RAND() * 10000, 2),
            DATE_SUB(CURDATE(), INTERVAL FLOOR(RAND() * 1095) DAY)
        );
        SET i = i + 1;
    END WHILE;
END//
DELIMITER ;

CALL generate_clients(1000);
DROP PROCEDURE generate_clients;"
```

---

##  **PASO 5: VERIFICACIÓN FINAL**

```powershell
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "VERIFICACIÓN FINAL DE REGISTROS" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan

Write-Host "`nPostgreSQL:" -ForegroundColor Green
docker exec -it postgres_gamlp psql -U admin -d negocios_db -c "SELECT COUNT(*) FROM proveedores;" 2>$null

Write-Host "`nMySQL:" -ForegroundColor Green
docker exec -it mysql_gamlp mysql -u root -p'dd8)JBY)/2026' -e "SELECT COUNT(*) FROM negocios_db.clientes;" 2>$null

Write-Host "`nSQL Server:" -ForegroundColor Green
docker exec -it sqlserver_gamlp /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "dd8)JBY)/2026" -Q "SELECT COUNT(*) FROM productos;" -h -1 2>$null

Write-Host "`n========================================" -ForegroundColor Cyan
Write-Host " BASES DE DATOS LISTAS PARA NIFI" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
```

---

## 🔌 **PASO 6: DATOS DE CONEXIÓN PARA NIFI**

| Gestor | Host | Puerto | Usuario | Password | Base de datos | Tabla |
|--------|------|--------|---------|----------|---------------|-------|
| PostgreSQL | `192.168.1.X` | `5433` | `admin` | `dd8)JBY)/2026` | `negocios_db` | `proveedores` |
| MySQL | `192.168.1.X` | `3307` | `root` | `dd8)JBY)/2026` | `negocios_db` | `clientes` |
| SQL Server | `192.168.1.X` | `1434` | `sa` | `dd8)JBY)/2026` | `master` | `productos` |

---

##  **PASO 7: DETENER Y LIMPIAR (CUANDO YA NO SE NECESITE)**

```powershell
# Detener contenedores
docker-compose down

# Eliminar volúmenes (borra datos)
docker-compose down -v
```


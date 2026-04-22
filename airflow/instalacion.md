
# **GUÍA COMPLETA: INSTALACIÓN DE APACHE AIRFLOW 2.6.3 EN DEBIAN 10 (VERSIÓN FINAL CORREGIDA)**

##  **Requisitos Previos**
- Debian 10 (Buster)
- Python 3.7+ instalado
- PostgreSQL 12+ (opcional pero recomendado)
- Acceso a internet
- Usuario con permisos sudo

---

##  **PASO 1: INSTALAR DEPENDENCIAS DEL SISTEMA**

```bash
# Actualizar repositorios
sudo apt update

# Instalar dependencias esenciales
sudo apt install -y \
  python3 \
  python3-pip \
  python3-venv \
  python3-dev \
  build-essential \
  libssl-dev \
  libffi-dev \
  libkrb5-dev \
  krb5-config \
  krb5-user \
  libsasl2-dev \
  libsasl2-modules \
  libsasl2-modules-gssapi-mit \
  libpq-dev \
  postgresql-client \
  wget \
  curl \
  git

# Verificar versión de Python
python3 --version
```

---

##  **PASO 2: CREAR DIRECTORIO Y ENTORNO VIRTUAL**

```bash
# Crear directorio para Airflow
sudo mkdir -p /usr/local/airflow
sudo chown -R $USER:$USER /usr/local/airflow
cd /usr/local/airflow

# Crear entorno virtual
python3 -m venv airflow_venv

# Activar entorno virtual
source airflow_venv/bin/activate

# Actualizar pip
pip install --upgrade pip setuptools wheel
```

---

##  **PASO 3: CONFIGURAR VARIABLES DE ENTORNO**

```bash
# Definir AIRFLOW_HOME
export AIRFLOW_HOME=/usr/local/airflow
echo 'export AIRFLOW_HOME=/usr/local/airflow' >> ~/.bashrc

# Definir URL de constraints (Python 3.7)
export CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-2.6.3/constraints-3.7.txt"

# Hacer permanente CONSTRAINT_URL (útil para futuras instalaciones)
echo "export CONSTRAINT_URL=\"${CONSTRAINT_URL}\"" >> ~/.bashrc
source ~/.bashrc
```

---

##  **PASO 4: INSTALAR APACHE AIRFLOW CORE**

```bash
# Instalar Apache Airflow
pip install apache-airflow==2.6.3 --constraint "${CONSTRAINT_URL}"

# Verificar instalación
airflow version
```

---

##  **PASO 5: INSTALAR PROVIDERS PARA HADOOP ECOSYSTEM**

```bash
# Instalar providers (excepto Kafka primero)
pip install \
  "apache-airflow-providers-apache-hdfs" \
  "apache-airflow-providers-apache-hive" \
  "apache-airflow-providers-http" \
  --constraint "${CONSTRAINT_URL}"

# Instalar confluent-kafka (versión compatible con Python 3.7)
pip install confluent-kafka==1.9.2

# Instalar provider Kafka sin dependencias
pip install apache-airflow-providers-apache-kafka --no-deps

# Verificar providers instalados
airflow providers list
```

---

##  **PASO 6: CONFIGURAR POSTGRESQL PARA AIRFLOW**

```bash
# Crear usuario y base de datos en PostgreSQL
sudo -u postgres psql <<EOF
CREATE USER airflow_user WITH PASSWORD 'airflow_password';
CREATE DATABASE airflow_db;
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO airflow_user;
EOF

# Instalar driver PostgreSQL para Python
pip install psycopg2-binary
```

---

##  **PASO 7: INICIALIZAR AIRFLOW**

```bash
# Inicializar la base de datos (crea airflow.cfg)
airflow db init

# Respaldar configuración original
cp $AIRFLOW_HOME/airflow.cfg $AIRFLOW_HOME/airflow.cfg.backup
```

---

##  **PASO 8: CONFIGURACIÓN AUTOMÁTICA DE AIRFLOW (SIN EDITAR MANUALMENTE)**

```bash
# Aplicar TODA la configuración con sed (automático)
sed -i 's|^sql_alchemy_conn = sqlite:///.*|sql_alchemy_conn = postgresql+psycopg2://airflow_user:airflow_password@localhost:5432/airflow_db|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|^executor = .*|executor = LocalExecutor|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|^load_examples = .*|load_examples = False|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|^authenticate = .*|authenticate = True|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|^auth_backend = .*|auth_backend = airflow.api.auth.backend.basic_auth|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|^base_log_folder = .*|base_log_folder = /usr/local/airflow/logs|' $AIRFLOW_HOME/airflow.cfg

#  CORRECCIÓN CLAVE: Cambiar puerto a 8081 (8080 suele estar ocupado)
sed -i '/^\[webserver\]/a web_server_port = 8081' $AIRFLOW_HOME/airflow.cfg

# Verificar cambios aplicados
echo "=== CONFIGURACIÓN APLICADA ==="
grep -E "sql_alchemy_conn|executor|load_examples|authenticate|auth_backend|base_log_folder|web_server_port" $AIRFLOW_HOME/airflow.cfg | grep -v "^#" | grep -v "^$"
```

---

## **PASO 9: REINICIALIZAR CON POSTGRESQL**

```bash
# Reinicializar la base de datos con la nueva configuración
airflow db reset -y
airflow db init
```

---

## **PASO 10: CREAR USUARIO ADMINISTRADOR**
```bash
# Crear usuario admin
airflow users create \
  --username admin \
  --firstname Admin \
  --lastname User \
  --role Admin \
  --email admin@example.com \
  --password admin123

# Verificar usuario creado
airflow users list
```

---

##  **PASO 11: CREAR SERVICIOS SYSTEMD (CON PUERTO 8081)**

### **11.1 Servicio para Airflow Webserver**
```bash
sudo tee /etc/systemd/system/airflow-webserver.service > /dev/null <<EOF
[Unit]
Description=Airflow Webserver
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=$USER
Group=$USER
Environment="PATH=/usr/local/airflow/airflow_venv/bin:/usr/local/bin:/usr/bin:/bin"
Environment="AIRFLOW_HOME=/usr/local/airflow"
WorkingDirectory=/usr/local/airflow
ExecStart=/usr/local/airflow/airflow_venv/bin/airflow webserver -p 8081
Restart=on-failure
RestartSec=10
KillMode=mixed
StandardOutput=append:/usr/local/airflow/logs/webserver.log
StandardError=append:/usr/local/airflow/logs/webserver_error.log

[Install]
WantedBy=multi-user.target
EOF
```

### **11.2 Servicio para Airflow Scheduler**
```bash
sudo tee /etc/systemd/system/airflow-scheduler.service > /dev/null <<EOF
[Unit]
Description=Airflow Scheduler
After=network.target postgresql.service airflow-webserver.service
Wants=postgresql.service

[Service]
Type=simple
User=$USER
Group=$USER
Environment="PATH=/usr/local/airflow/airflow_venv/bin:/usr/local/bin:/usr/bin:/bin"
Environment="AIRFLOW_HOME=/usr/local/airflow"
WorkingDirectory=/usr/local/airflow
ExecStart=/usr/local/airflow/airflow_venv/bin/airflow scheduler
Restart=always
RestartSec=10
KillMode=mixed
StandardOutput=append:/usr/local/airflow/logs/scheduler.log
StandardError=append:/usr/local/airflow/logs/scheduler_error.log

[Install]
WantedBy=multi-user.target
EOF
```

---

##  **PASO 12: ACTIVAR SERVICIOS**

```bash
# Recargar configuración de systemd
sudo systemctl daemon-reload

# Habilitar servicios para inicio automático
sudo systemctl enable airflow-webserver
sudo systemctl enable airflow-scheduler

# Iniciar servicios
sudo systemctl start airflow-webserver
sudo systemctl start airflow-scheduler

# Esperar 5 segundos a que inicien
sleep 5

# Verificar estado
sudo systemctl status airflow-webserver --no-pager
sudo systemctl status airflow-scheduler --no-pager
```

---

## **PASO 13: CONFIGURAR FIREWALL**

```bash
# Abrir puerto 8081 en el firewall
sudo ufw allow 8081/tcp
sudo ufw reload

# Verificar que el puerto está escuchando
ss -tlnp | grep 8081
```

---

##  **PASO 14: VERIFICACIÓN FINAL Y ACCESO**

```bash
# Obtener IP del servidor
IP_SERVIDOR=$(hostname -I | awk '{print $1}')

# Verificar que Airflow responde
curl -I http://localhost:8081/login/

# Mostrar información de acceso
echo "========================================"
echo " AIRFLOW INSTALADO CORRECTAMENTE"
echo "========================================"
echo "URL: http://${IP_SERVIDOR}:8081/login/"
echo "Usuario: admin"
echo "Password: admin123"
echo "========================================"
```

---

##  **PASO 15: CREAR PRIMER DAG DE PRUEBA**

```bash
# Crear directorio para DAGs (si no existe)
mkdir -p $AIRFLOW_HOME/dags

# Crear DAG de ejemplo
cat > $AIRFLOW_HOME/dags/mi_primer_dag.py <<'EOF'
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

default_args = {
    'owner': 'admin',
    'depends_on_past': False,
    'start_date': datetime(2026, 4, 21),
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

def mi_funcion_python():
    print("¡Hola desde Airflow en Debian 10!")
    return "Ejecución exitosa"

with DAG(
    'mi_primer_dag',
    default_args=default_args,
    description='Mi primer DAG en Airflow',
    schedule_interval='@daily',
    catchup=False,
    tags=['prueba'],
) as dag:
    
    tarea_inicio = BashOperator(
        task_id='inicio',
        bash_command='echo "Iniciando DAG - $(date)"',
    )
    
    tarea_python = PythonOperator(
        task_id='tarea_python',
        python_callable=mi_funcion_python,
    )
    
    tarea_fin = BashOperator(
        task_id='fin',
        bash_command='echo "DAG completado - $(date)"',
    )
    
    tarea_inicio >> tarea_python >> tarea_fin
EOF

echo " DAG de prueba creado en: $AIRFLOW_HOME/dags/mi_primer_dag.py"
```

---

##  **COMANDOS ÚTILES PARA GESTIÓN**

```bash
# Ver logs de servicios
sudo journalctl -u airflow-webserver --since today
sudo journalctl -u airflow-scheduler --since today

# Reiniciar servicios
sudo systemctl restart airflow-webserver
sudo systemctl restart airflow-scheduler

# Detener servicios
sudo systemctl stop airflow-webserver
sudo systemctl stop airflow-scheduler

# Ver estado de la base de datos
airflow db check

# Listar DAGs disponibles
airflow dags list

# Probar una tarea específica
airflow tasks test mi_primer_dag tarea_python 2026-04-21
```

---

##  **SOLUCIÓN DE PROBLEMAS COMUNES**

### **Error: Puerto 8080 ocupado**
```bash
# YA CORREGIDO: Usamos puerto 8081 por defecto
```

### **Error: "airflow: command not found"**
```bash
source /usr/local/airflow/airflow_venv/bin/activate
```

### **Error: 404 Not Found en /**
```bash
# Usar URL con /login/
curl http://localhost:8081/login/
```

### **Error: Conexión rechazada a PostgreSQL**
```bash
sudo systemctl status postgresql
sudo -u postgres psql -c "SELECT usename FROM pg_user;"
```

---

##  **ESTRUCTURA FINAL**

```
/usr/local/airflow/
├── airflow_venv/          # Entorno virtual Python
├── airflow.cfg            # Configuración (puerto 8081)
├── airflow.db             # No usado (usamos PostgreSQL)
├── dags/                  # Directorio de DAGs
├── logs/                  # Logs de ejecución
└── webserver_config.py    # Configuración del webserver
```

---

##  **CHECKLIST DE VERIFICACIÓN**

```bash
echo "=== VERIFICACIÓN COMPLETA ==="
echo "✓ Airflow: $(airflow version)"
echo "✓ Webserver: $(systemctl is-active airflow-webserver)"
echo "✓ Scheduler: $(systemctl is-active airflow-scheduler)"
echo "✓ Puerto 8081: $(ss -tlnp | grep 8081 | wc -l) escuchando"
echo "✓ PostgreSQL: $(sudo -u postgres psql -t -c "SELECT 1 FROM pg_database WHERE datname='airflow_db'" | xargs)"
echo "✓ Usuario admin: $(airflow users list | grep admin | wc -l) encontrado"
echo "✓ URL: http://$(hostname -I | awk '{print $1}'):8081/login/"
```

---

##  **ACCESO FINAL**

```
URL: http://<IP_DEL_SERVIDOR>:8081/login/
Usuario: admin
Password: admin123
```

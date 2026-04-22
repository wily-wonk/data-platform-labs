
# Guía de Instalación: Apache Airflow 2.6.3

Esta guía detalla la instalación de Apache Airflow en Debian, utilizando Python 3.7. Se aplica el principio de mínimo privilegio creando un usuario dedicado y se configura PostgreSQL como base de datos de metadatos, junto con los conectores necesarios para interactuar con Hadoop, Hive y Kafka.

### PASO 1: INSTALAR DEPENDENCIAS DEL SISTEMA
**Usuario requerido:** Administrador (sudo)

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

### PASO 2: CREAR USUARIO DEDICADO, DIRECTORIO Y ENTORNO VIRTUAL
**Usuario requerido:** Administrador (sudo)

```bash
# Crear grupo y usuario de servicio sin privilegios root
sudo groupadd airflow
sudo useradd -r -m -d /usr/local/airflow -s /bin/bash -g airflow airflow

# Cambiar a la sesión del nuevo usuario para las instalaciones
sudo su - airflow

# Crear entorno virtual
python3 -m venv airflow_venv

# Activar entorno virtual
source airflow_venv/bin/activate

# Actualizar pip
pip install --upgrade pip setuptools wheel
```

### PASO 3: CONFIGURAR VARIABLES DE ENTORNO
**Usuario requerido:** `airflow`

```bash
# Definir AIRFLOW_HOME
export AIRFLOW_HOME=/usr/local/airflow
echo 'export AIRFLOW_HOME=/usr/local/airflow' >> ~/.bashrc

# Definir URL de constraints (Para Python 3.7)
export CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-2.6.3/constraints-3.7.txt"
echo "export CONSTRAINT_URL=\"${CONSTRAINT_URL}\"" >> ~/.bashrc
source ~/.bashrc
```

### PASO 4: INSTALAR APACHE AIRFLOW CORE
**Usuario requerido:** `airflow` (con entorno virtual activado)

```bash
# Instalar Apache Airflow
pip install apache-airflow==2.6.3 --constraint "${CONSTRAINT_URL}"

# Verificar instalación
airflow version
```

### PASO 5: INSTALAR PROVIDERS PARA HADOOP ECOSYSTEM
**Usuario requerido:** `airflow` (con entorno virtual activado)

```bash
# Instalar providers base
pip install \
  "apache-airflow-providers-apache-hdfs" \
  "apache-airflow-providers-apache-hive" \
  "apache-airflow-providers-http" \
  --constraint "${CONSTRAINT_URL}"

# Instalar confluent-kafka (versión compatible con Python 3.7)
pip install confluent-kafka==1.9.2

# Instalar provider Kafka sin dependencias cruzadas
pip install apache-airflow-providers-apache-kafka --no-deps

# Verificar providers instalados
airflow providers list
```

### PASO 6: CONFIGURAR POSTGRESQL PARA AIRFLOW
**Usuario requerido:** Salir de `airflow` (usar `exit`) y ejecutar como sudoer.

```bash
# Crear usuario y base de datos en PostgreSQL
sudo -u postgres psql <<EOF
CREATE USER airflow_user WITH PASSWORD 'airflow_password';
CREATE DATABASE airflow_db;
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO airflow_user;
EOF

# Volver al usuario airflow e instalar el driver de BD
sudo su - airflow
source airflow_venv/bin/activate
pip install psycopg2-binary
```

### PASO 7: INICIALIZAR Y CONFIGURAR AIRFLOW
**Usuario requerido:** `airflow` (con entorno virtual activado)

```bash
# Inicializar la base de datos sqlite temporal (crea la estructura de carpetas y airflow.cfg)
airflow db init

# Respaldar configuración original
cp $AIRFLOW_HOME/airflow.cfg $AIRFLOW_HOME/airflow.cfg.backup

# Aplicar TODA la configuración mediante comandos nativos seguros
airflow config set core sql_alchemy_conn postgresql+psycopg2://airflow_user:airflow_password@localhost:5432/airflow_db
airflow config set core executor LocalExecutor
airflow config set core load_examples False
airflow config set webserver authenticate True
airflow config set api auth_backends airflow.api.auth.backend.basic_auth
airflow config set logging base_log_folder /usr/local/airflow/logs

# Cambiar puerto a 8081 para evitar conflictos con otros monitores
airflow config set webserver web_server_port 8081
```

### PASO 8: REINICIALIZAR CON POSTGRESQL Y CREAR ADMIN
**Usuario requerido:** `airflow` (con entorno virtual activado)

```bash
# Migrar la base de datos oficial a PostgreSQL
airflow db init

# Crear usuario admin
airflow users create \
  --username admin \
  --firstname Admin \
  --lastname User \
  --role Admin \
  --email admin@gamlp.gob.bo \
  --password admin123

# Verificar usuario creado
airflow users list
```

### PASO 9: CREAR SERVICIOS SYSTEMD (CON PUERTO 8081)
**Usuario requerido:** Salir de `airflow` (usar `exit`) y ejecutar como sudoer.

**9.1 Servicio para Airflow Webserver**
```bash
sudo tee /etc/systemd/system/airflow-webserver.service > /dev/null <<EOF
[Unit]
Description=Airflow Webserver
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=airflow
Group=airflow
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

**9.2 Servicio para Airflow Scheduler**
```bash
sudo tee /etc/systemd/system/airflow-scheduler.service > /dev/null <<EOF
[Unit]
Description=Airflow Scheduler
After=network.target postgresql.service airflow-webserver.service
Wants=postgresql.service

[Service]
Type=simple
User=airflow
Group=airflow
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

### PASO 10: ACTIVAR SERVICIOS Y CONFIGURAR FIREWALL
**Usuario requerido:** Administrador (sudo)

```bash
# Recargar configuración de systemd y habilitar servicios
sudo systemctl daemon-reload
sudo systemctl enable airflow-webserver airflow-scheduler
sudo systemctl start airflow-webserver airflow-scheduler

# Abrir puerto 8081 en el firewall
sudo ufw allow 8081/tcp
sudo ufw reload

# Esperar 5 segundos a que inicien y verificar estado
sleep 5
sudo systemctl status airflow-webserver --no-pager
sudo systemctl status airflow-scheduler --no-pager
```

### PASO 11: VERIFICACIÓN FINAL Y CREACIÓN DE DAG
**Usuario requerido:** Salir a tu usuario normal y ejecutar comandos de red.

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

**Crear DAG de Ejemplo (Como usuario `airflow`):**
```bash
sudo su - airflow
mkdir -p $AIRFLOW_HOME/dags

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
    print("¡Hola desde Airflow en Debian!")
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
```

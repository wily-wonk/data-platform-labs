
# Guía de Instalación: Apache Airflow 2.6.3

> **Entorno:** Debian 10 | Python 3.7 | PostgreSQL 11 | Kafka 3.7.2 | HDFS 3.4.0 | Hive 4.0.0
> **Puerto:** 8081 (sin conflicto con Kafka UI en 8080)
> **Usuario de servicio:** `airflow`

---

## PASO 1: INSTALAR DEPENDENCIAS DEL SISTEMA

**Usuario:** Administrador (sudo)

```bash
sudo apt update

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
  librdkafka-dev \
  postgresql-client \
  wget \
  curl \
  git

python3 --version
```

---

## PASO 2: CREAR USUARIO DEDICADO, DIRECTORIO Y ENTORNO VIRTUAL

**Usuario:** Administrador (sudo)

```bash
sudo groupadd airflow
sudo useradd -r -m -d /usr/local/airflow -s /bin/bash -g airflow airflow

sudo su - airflow

python3 -m venv airflow_venv
source airflow_venv/bin/activate
pip install --upgrade pip setuptools wheel
```

---

## PASO 3: CONFIGURAR VARIABLES DE ENTORNO

**Usuario:** airflow (con entorno virtual activado)

```bash
export AIRFLOW_HOME=/usr/local/airflow
echo 'export AIRFLOW_HOME=/usr/local/airflow' >> ~/.bashrc

export CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-2.6.3/constraints-3.7.txt"
echo "export CONSTRAINT_URL=\"${CONSTRAINT_URL}\"" >> ~/.bashrc

source ~/.bashrc
```

---

## PASO 4: INSTALAR APACHE AIRFLOW CORE

**Usuario:** airflow (con entorno virtual activado)

```bash
pip install apache-airflow==2.6.3 --constraint "${CONSTRAINT_URL}"
airflow version
```

---

## PASO 5: INSTALAR PROVIDERS PARA ECOSISTEMA HADOOP

**Usuario:** airflow (con entorno virtual activado)

```bash
# Providers base
pip install \
  "apache-airflow-providers-apache-hdfs" \
  "apache-airflow-providers-apache-hive" \
  "apache-airflow-providers-http" \
  --constraint "${CONSTRAINT_URL}"

# Provider Kafka - compatible Python 3.7 + librdkafka-dev
pip install confluent-kafka==1.9.2
pip install apache-airflow-providers-apache-kafka --no-deps

airflow providers list
```

---

## PASO 6: CONFIGURAR POSTGRESQL PARA AIRFLOW

**Usuario:** Salir de airflow (`exit`) y ejecutar como sudoer

```bash
sudo -u postgres psql <<EOF
CREATE USER airflow_user WITH PASSWORD '[PASSWORD]';
CREATE DATABASE airflow_db;
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO airflow_user;
EOF

sudo su - airflow
source airflow_venv/bin/activate

# Versión específica compatible con Debian 10 (glibc 2.28)
pip install psycopg2-binary==2.9.5
```

---

## PASO 7: INICIALIZAR Y CONFIGURAR AIRFLOW

**Usuario:** airflow (con entorno virtual activado)

```bash
# Generar airflow.cfg sin crear base de datos SQLite
airflow config list > /dev/null
cp $AIRFLOW_HOME/airflow.cfg $AIRFLOW_HOME/airflow.cfg.backup

# Aplicar configuración con sed
sed -i 's|sql_alchemy_conn = sqlite:////usr/local/airflow/airflow.db|sql_alchemy_conn = postgresql+psycopg2://airflow_user:[PASSWORD]@localhost:5432/airflow_db|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|executor = SequentialExecutor|executor = LocalExecutor|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|load_examples = True|load_examples = False|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|web_server_port = 8080|web_server_port = 8081|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|^authenticate = False|authenticate = True|' $AIRFLOW_HOME/airflow.cfg
sed -i 's|auth_backends = airflow.api.auth.backend.session|auth_backends = airflow.api.auth.backend.basic_auth|' $AIRFLOW_HOME/airflow.cfg

# Verificar cambios
grep -E "sql_alchemy_conn|executor =|load_examples|web_server_port|^authenticate|auth_backends" $AIRFLOW_HOME/airflow.cfg
```

---

## PASO 8: INICIALIZAR BASE DE DATOS Y CREAR ADMIN

**Usuario:** airflow (con entorno virtual activado)

```bash
# Inicializar la base de datos en PostgreSQL
airflow db init

# Crear usuario administrador
airflow users create \
  --username admin \
  --firstname Admin \
  --lastname User \
  --role Admin \
  --email [EMAIL] \
  --password '[PASSWORD]'

# Verificar usuario creado
airflow users list
```

---

## PASO 9: CREAR SERVICIOS SYSTEMD

**Usuario:** Salir de airflow (`exit`) y ejecutar como sudoer

### 9.1 Airflow Webserver

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

[Install]
WantedBy=multi-user.target
EOF
```

### 9.2 Airflow Scheduler

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

[Install]
WantedBy=multi-user.target
EOF
```

---

## PASO 10: ACTIVAR SERVICIOS Y CONFIGURAR FIREWALL

**Usuario:** Administrador (sudo)

```bash
sudo systemctl daemon-reload
sudo systemctl enable airflow-webserver airflow-scheduler
sudo systemctl start airflow-webserver airflow-scheduler

sudo ufw allow 8081/tcp
sudo ufw reload

sleep 5
sudo systemctl status airflow-webserver --no-pager
sudo systemctl status airflow-scheduler --no-pager
```

---

## VERIFICACIÓN Y ACCESO

```bash
# Ver logs en tiempo real
sudo journalctl -u airflow-webserver -f
sudo journalctl -u airflow-scheduler -f

# Verificar puerto
ss -tlnp | grep 8081
```

- **URL:** `http://<IP_DEL_SERVIDOR>:8081`
- **Usuario:** `admin`
- **Contraseña:** la definida en el Paso 8

---

Guárdala como `AIRFLOW_INSTALL.md`. ¿Algo más que ajustar?

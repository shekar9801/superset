#!/bin/bash

# 1. Update System
sudo apt update -y && sudo apt upgrade -y

# 2. Install Dependencies
sudo apt-get install build-essential libssl-dev libffi-dev python3-dev python3-pip \
libsasl2-dev libldap2-dev default-libmysqlclient-dev python3.10-venv -y

# 3. Create App Directory for Superset and Dependencies
sudo mkdir /app
sudo chown $USER /app
cd /app

# 4. Create Python Environment
mkdir superset
cd superset
python3 -m venv superset_env
source superset_env/bin/activate
pip install --upgrade setuptools pip

# 5. Install Required Dependencies
pip install pillow
pip install apache-superset

# 6. Create Superset Config File and Set Environment Variable
touch superset_config.py
export SUPERSET_CONFIG_PATH=/app/superset/superset_config.py

# Generate SECRET_KEY
SECRET_KEY=$(openssl rand -base64 42)

# Add configuration to superset_config.py
echo "
# Superset specific config
ROW_LIMIT = 5000

# Flask App Builder configuration
SECRET_KEY = '$SECRET_KEY'

# SQLAlchemy connection string to the database storing Superset metadata
SQLALCHEMY_DATABASE_URI = 'sqlite:////app/superset/superset.db?check_same_thread=false'

TALISMAN_ENABLED = False
WTF_CSRF_ENABLED = False

# Mapbox API Key
MAPBOX_API_KEY = ''
" > superset_config.py

# 7. Initialize Superset Database
export FLASK_APP=superset

# Upgrade the database
superset db upgrade

# Create admin user (modify as needed)
superset fab create-admin

# Initialize default roles and permissions
superset init

# 8. Create Superset Startup Script
echo "
#!/bin/bash
export SUPERSET_CONFIG_PATH=/app/superset/superset_config.py
source /app/superset/superset_env/bin/activate
gunicorn \\
      -w 10 \\
      -k gevent \\
      --timeout 120 \\
      -b  0.0.0.0:8088 \\
      --limit-request-line 0 \\
      --limit-request-field_size 0 \\
      --statsd-host localhost:8125 \\
      'superset.app:create_app()'
" > run_superset.sh

# Grant execute permission
chmod +x run_superset.sh

# 9. Create Systemd Service for Superset
sudo bash -c "cat > /etc/systemd/system/superset.service <<EOF
[Unit]
Description=Apache Superset Webserver Daemon
After=network.target

[Service]
PIDFile=/app/superset/superset-webserver.PIDFile
Environment=SUPERSET_HOME=/app/superset
Environment=PYTHONPATH=/app/superset
WorkingDirectory=/app/superset
ExecStart=/app/superset/run_superset.sh
ExecStop=/bin/kill -s TERM \$MAINPID

[Install]
WantedBy=multi-user.target
EOF"

# Reload systemd, enable, and start Superset service
sudo systemctl daemon-reload
sudo systemctl enable superset.service
sudo systemctl start superset.service

# Check status of the Superset service
sudo systemctl status superset.service

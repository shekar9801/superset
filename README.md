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

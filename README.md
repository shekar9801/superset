## SUPERSET PRODUCTION INSTALLATION


#### Set up environment

* Update Ubuntu/Debian

```
sudo apt update -y & sudo apt upgrade -y
```
* Install dependencies

```
sudo apt-get install build-essential libssl-dev libffi-dev python3-dev python3-pip libsasl2-dev libldap2-dev default-libmysqlclient-dev python3.10-venv
``` 

* Create app directory for superset and dependencies 

```
sudo mkdir /app
sudo chown user /app
cd /app
```

* Create python environment 

```
mkdir superset
cd superset
python3 -m venv superset_env
. superset_env/bin/activate
pip install --upgrade setuptools pip
```

* Install Required dependencies

```
pip install pillow
pip install apache-superset
```


* Create superset config file and set environment variable 

```
touch superset_config.py
export SUPERSET_CONFIG_PATH=/app/superset/superset_config.py

```

* Edit and paste following code in it

```
# Superset specific config
ROW_LIMIT = 5000

# Flask App Builder configuration
# Your App secret key will be used for securely signing the session cookie
# and encrypting sensitive information on the database
# Make sure you are changing this key for your deployment with a strong key.
# Alternatively you can set it with `SUPERSET_SECRET_KEY` environment variable.
# You MUST set this for production environments or the server will not refuse
# to start and you will see an error in the logs accordingly.
SECRET_KEY = 'YOUR_OWN_RANDOM_GENERATED_SECRET_KEY'

# The SQLAlchemy connection string to your database backend
# This connection defines the path to the database that stores your
# superset metadata (slices, connections, tables, dashboards, ...).
# Note that the connection information to connect to the datasources
# you want to explore are managed directly in the web UI
# The check_same_thread=false property ensures the sqlite client does not attempt
# to enforce single-threaded access, which may be problematic in some edge cases
SQLALCHEMY_DATABASE_URI = 'sqlite:////app/superset/superset.db?check_same_thread=false'

TALISMAN_ENABLED = False
WTF_CSRF_ENABLED = False

# Set this API key to enable Mapbox visualizations
MAPBOX_API_KEY = ''
```

Please replace YOUR_OWN_RANDOM_GENERATED_SECRET_KEY in above file with the code returned by following command

```
openssl rand -base64 42
```

* Once Done let us inititlize database with following commands 

```
# Create an admin user in your metadata database (use `admin` as username to be able to load the examples)
export FLASK_APP=superset

superset db upgrade

superset fab create-admin

# As this is going to be production I have commented load example part but if you need you can run this
# superset load_examples

# Create default roles and permissions
superset init

```

* Now Our environment is ready lets try running it..
To run superset I have created a sh script that you can run in order to run the server. To create create script using following command.

```
nano run_superset.sh
```

and paste following code in it.

```
#!/bin/bash
export SUPERSET_CONFIG_PATH=/app/superset/superset_config.py
 . /app/superset/superset_env/bin/activate
gunicorn \
      -w 10 \
      -k gevent \
      --timeout 120 \
      -b  0.0.0.0:8088 \
      --limit-request-line 0 \
      --limit-request-field_size 0 \
      --statsd-host localhost:8125 \
      "superset.app:create_app()"
```


* In order to run it we need to grant it run permission. To do that lets run following command.
```
chmod +x run_superset.sh
```

 * Lets run and test if it works?

```
sh run_superset.sh
```

* check if you are able to login using admin creds on server-ip-address:8088. If everything is working fine then we can go ahead and create service that will start automatically as soon as server starts or in case it reboots.

Lets create service called superset using following command

```
sudo nano /etc/systemd/system/superset.service
```

paste following code in it 

```
[Unit]
Description = Apache Superset Webserver Daemon
After = network.target

[Service]
PIDFile = /app/superset/superset-webserver.PIDFile
Environment=SUPERSET_HOME=/app/superset
Environment=PYTHONPATH=/app/superset
WorkingDirectory = /app/superset
limit-re>
ExecStart = /app/superset/run_superset.sh
ExecStop = /bin/kill -s TERM $MAINPID


[Install]
WantedBy=multi-user.target

```

once copied run following command to enable and start service

```
systemctl daemon-reload
sudo systemctl enable superset.service
sudo systemctl start superset.service
```
For connecting or integrating the Apache pinot with Apache Superset.
```
pinot://172.25.0.5:8000/query/sql?controller=http://172.25.0.5:9000
```

To stop, remove and clear the running docker containers.
```
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker system prune -a -f --volumes
```
Schema for the creating a OFFLINE table in the Apache Pinot.
```
{
    "schemaName": "energy_production_schema",
    "dimensionFieldSpecs": [
      {
        "name": "serial_num",
        "dataType": "STRING",
        "defaultNullValue": "N/A"
      },
      {
        "name": "manufacturer",
        "dataType": "STRING",
        "defaultNullValue": "N/A"
      },
      {
        "name": "installationDate",
        "dataType": "STRING",
        "defaultNullValue": "N/A"
      }
    ],
    "metricFieldSpecs": [
      {
        "name": "ac_voltage",
        "dataType": "FLOAT",
        "defaultNullValue": 0.0
      },
      {
        "name": "ac_frequency",
        "dataType": "FLOAT",
        "defaultNullValue": 0.0
      },
      {
        "name": "dc_voltage",
        "dataType": "FLOAT",
        "defaultNullValue": 0.0
      },
      {
        "name": "dc_current",
        "dataType": "FLOAT",
        "defaultNullValue": 0.0
      },
      {
        "name": "energy_produced",
        "dataType": "FLOAT",
        "defaultNullValue": 0.0
      }
    ],
    "dateTimeFieldSpecs": [
      {
        "name": "timestamp",
        "dataType": "LONG",
        "format": "1:MILLISECONDS:EPOCH",
        "granularity": "1:MILLISECONDS"
      }
    ]
  }
  ```
Configuration for the shema.
```
{
    "tableName": "energy_production_OFFLINE",
    "tableType": "OFFLINE",
    "quota": {
      "maxQueriesPerSecond": 300,
      "storage": "140G"
    },
    "routing": {
      "segmentPrunerTypes": ["partition"],
      "instanceSelectorType": "replicaGroup"
    },
    "segmentsConfig": {
      "timeColumnName": "timestamp",
      "timeType": "MILLISECONDS",
      "schemaName": "energy_production_schema",
      "replication": 1,
      "segmentsRetention": "1d"
    },
    "tenants": {
      "broker": "DefaultTenant",
      "server": "DefaultTenant"
    },
    "tableIndexConfig": {
      "loadMode": "MMAP",
      "noDictionaryColumns": [],
      "enableDefaultStarTree": false,
      "enableDynamicStarTreeCreation": false,
      "aggregateMetrics": false,
      "nullHandlingEnabled": false,
      "autoGeneratedInvertedIndex": false,
      "createInvertedIndexDuringSegmentGeneration": false,
      "streamConfigs": {
        "streamType": "kafka",
        "stream.kafka.consumer.type": "simple",
        "stream.kafka.topic.name": "energy_data",
        "stream.kafka.decoder.class.name": "org.apache.pinot.plugin.stream.kafka.KafkaJSONMessageDecoder",
        "stream.kafka.consumer.factory.class.name": "org.apache.pinot.plugin.stream.kafka20.KafkaConsumerFactory",
        "stream.kafka.broker.list": "PLAINTEXT://13.52.83.241:9092",
        "realtime.segment.flush.threshold.time": "12h",
        "realtime.segment.flush.threshold.size": "100000",
        "stream.kafka.consumer.prop.auto.offset.reset": "smallest"
      }
    },
    "metadata": {
      "customConfigs": {}
    },
    "fieldConfigList": [],
    "ingestionConfig": {
      "filterConfig": {},
      "transformConfigs": [
        {
          "columnName": "serial_num",
          "transformFunction": "jsonFormat(serialNum)"
        },
        {
          "columnName": "manufacturer",
          "transformFunction": "jsonFormat(details.manufacturer)"
        },
        {
          "columnName": "installationDate",
          "transformFunction": "jsonFormat(details.installationDate)"
        },
        {
          "columnName": "ac_voltage",
          "transformFunction": "jsonFormat(acVoltage)"
        },
        {
          "columnName": "ac_frequency",
          "transformFunction": "jsonFormat(acFrequency)"
        },
        {
          "columnName": "dc_voltage",
          "transformFunction": "jsonFormat(dcVoltage)"
        },
        {
          "columnName": "dc_current",
          "transformFunction": "jsonFormat(dcCurrent)"
        },
        {
          "columnName": "energy_produced",
          "transformFunction": "jsonFormat(energyProduced)"
        }
      ]
    }
  }
  ```
To create the segments and upload to the apache pinot table.
```
#!/bin/bash

# Step 1: Copy segment files and configuration to the Docker container
docker cp /home/ubuntu/work/Enphase_data/kafka-pinot/py/ubuntu/work/Enphase_data/kafka-pinot/data/segments pinot:/tmp/energy_data
docker cp /home/ubuntu/work/Enphase_data/kafka-pinot/pinot/def-schema.json pinot:/tmp/def-schema.json
docker cp /home/ubuntu/work/Enphase_data/kafka-pinot/pinot/def-table.json pinot:/tmp/def-table.json

# Step 2: Create segments inside the Docker container
create_segment_command="/opt/pinot/bin/pinot-admin.sh CreateSegment \
  -dataDir /tmp/energy_data \
  -format JSON \
  -outDir /tmp/segments \
  -overwrite \
  -tableConfigFile /tmp/def-table.json \
  -schemaFile /tmp/def-schema.json"

echo "Executing command: $create_segment_command"
docker exec pinot bash -c "$create_segment_command"

# Step 3: Upload the created segments to Pinot
upload_segment_command="/opt/pinot/bin/pinot-admin.sh UploadSegment \
  -controllerHost 13.52.83.241 \
  -controllerPort 9000 \
  -segmentDir /tmp/segments \
  -tableName energy_production_REALTIME"

echo "Executing command: $upload_segment_command"
docker exec pinot bash -c "$upload_segment_command"
```
```
pip install --upgrade $(pip list --outdated | awk 'NR>2 {print $1}')
```


# Batch Ingestion into the influxdb

```

# Load environment variables
import os
import numpy as np
from influxdb_client import InfluxDBClient, Point, WritePrecision  # type: ignore
from influxdb_client.client.write_api import SYNCHRONOUS  # type: ignore
import pandas as pd

# Set display options for pandas
pd.set_option('display.max_columns', None)

# Load environment variables
token = os.getenv("INFLUXDB_TOKEN", "UXHZ0-LEw4O9wBEVm_TpODVBDit6x86fQmj87y5Fuq7Mgaz0vrZhG8ponUap5DUshmWfFfiG8pQBBeAJ4IZIdQ==")

org = "IISc"
url = "http://13.52.83.241:8086"
bucket = "sensor_data"

# Initialize InfluxDB client with increased timeout
client = InfluxDBClient(url=url, token=token, org=org, timeout=30_000)  # 30 seconds

# Ensure the bucket exists
buckets_api = client.buckets_api()
existing_buckets = buckets_api.find_buckets().buckets
if not any(bucket.name == "sensor_data" for bucket in existing_buckets):
    buckets_api.create_bucket(bucket_name=bucket, org=org)

# Load the CSV data
columns = ['date', 'timestamp', 'site_id', 'serial_num', 'ac_voltage', 'ac_current',
           'ac_frequency', 'dc_voltage', 'dc_current', 'temperature', 'energy_produced']
data = pd.read_csv("C://Users//admin//OneDrive - Talentpace//15_site_data//telemetry//Site-4056397.csv", usecols=columns)

# Convert timestamp to UTC and then to int64 in nanoseconds
data['timestamp'] = pd.to_datetime(data['timestamp'], utc=True).astype(np.int64)

# Check for missing values
if data['timestamp'].isnull().any():
    print("Warning: Some timestamps are null after conversion.")

write_api = client.write_api(write_options=SYNCHRONOUS)

# Prepare a list to hold all the points
points = []

# Iterate over the DataFrame and create points
for index, row in data.iterrows():
    # Validate tag values
    if pd.isnull(row['site_id']) or pd.isnull(row['serial_num']):
        print(f"Warning: Missing tag values in row {index}. Skipping this row.")
        continue  # Skip this row if tags are missing

    # Validate and convert fields
    try:
        energy_produced = int(row['energy_produced']) if not pd.isnull(row['energy_produced']) else 0
        temperature = int(row['temperature']) if not pd.isnull(row['temperature']) else 0
    except ValueError as e:
        print(f"Error converting values in row {index}: {e}. Skipping this row.")
        continue

    # Create the point and append it to the list
    point = Point("sensor_readings") \
        .tag("site_id", str(row['site_id'])) \
        .tag("serial_num", str(row['serial_num'])) \
        .field("ac_voltage", row['ac_voltage']) \
        .field("ac_current", row['ac_current']) \
        .field("ac_frequency", row['ac_frequency']) \
        .field("dc_voltage", row['dc_voltage']) \
        .field("dc_current", row['dc_current']) \
        .field("temperature", temperature) \
        .field("energy_produced", energy_produced) \
        .time(row['timestamp'], WritePrecision.NS)

    points.append(point)

# Log the total number of points to be written
print(f"Total points to write: {len(points)}")

# Batch size for writing
batch_size = 1000  # Adjust based on your testing
for i in range(0, len(points), batch_size):
    batch_points = points[i:i + batch_size]
    try:
        write_api.write(bucket=bucket, org=org, record=batch_points)
        print(f"Wrote batch {i // batch_size + 1} with {len(batch_points)} points.")
    except Exception as e:
        print(f"Error writing batch {i // batch_size + 1}: {e}")

client.close()

```















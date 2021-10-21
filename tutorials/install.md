# BlueEngine install guide

## Installing

Install dependencies

Required dependencies:

```bash
sudo apt-get update
sudo apt-get install -y git \
                        build-essential \
                        gcc \
                        make \
                        yasm \
                        autoconf \
                        automake \
                        cmake \
                        libtool \
                        checkinstall \
                        pkg-config
```

If you want to use features like video or image IO, install the following:

```bash
sudo apt-get install -y --no-install-recommends libc6-dev \
libgdiplus software-properties-common ffmpeg

python3 -m pip install numpy
python3 -m pip install Cython
conda install -c conda-forge ffmpeg
conda install -c conda-forge opencv

python3 -m pip install ffmpeg-python
```

## Install the BlueEngine SDK:

Run the following script

```bash
# change the target version you want to use
TARGET_VERSION=master

echo "Installing from $TARGET_VERSION"
pip install --upgrade setuptools

python3 -m pip install git+ssh://git@github.com/liftupvn/blueengine.git@$TARGET_VERSION

pip install --upgrade setuptools

echo "==================================================="
echo "Installed blueengine SDK. Happy building engines ;D"
```

## Run the networking system on local machine

We use `Kafka` for communication between the components, and `Redis` for data storage and caching. On local machine, you can use the folowing `docker-compose.yml` file for easily setup.

```yaml
version: "3.8"

services:
    zoo1:
        restart: always
        image: zookeeper:3.4.9
        hostname: zoo1
        ports:
            - "2181:2181"
        environment:
            ZOO_MY_ID: 1
            ZOO_PORT: 2181
            ZOO_SERVERS: server.1=zoo1:2888:3888
        # volumes:
        #     - ./zk-single-kafka-single/zoo1/data:/data
        #     - ./zk-single-kafka-single/zoo1/datalog:/datalog

    kafka1:
        restart: always
        image: confluentinc/cp-kafka:5.5.1
        hostname: kafka1
        ports:
            - "9092:9092"
            - "9999:9999"
        environment:
            KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka1:19092,LISTENER_DOCKER_EXTERNAL://${HOST_IP:-127.0.0.1}:9092
            KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
            KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
            KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181"
            KAFKA_BROKER_ID: 1
            KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
            KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
            KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
            KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
            KAFKA_JMX_PORT: 9999
            KAFKA_JMX_HOSTNAME: ${HOST_IP:-127.0.0.1}
        # volumes:
        #     - ./zk-single-kafka-single/kafka1/data:/var/lib/kafka/data
        depends_on:
            - zoo1
    
    redis:
        restart: always
        image: "redis"
        ports:
            - 0.0.0.0:6379:6379
```

Also create the `.env` file that define the `HOST_IP` value, which is your host's IP, that is passed into the above `docker-compose.yml`.

```
HOST_IP=192.168.80.186
```

Create the file and then run:

```bash
docker-compose up -d
```

## Run the BlueEdge management services on local machine.

There are two main BlueEdge services must be online so you can start develop the engines.

- `Restful` service is like a API gateway that takes the incoming requests and register the `session` to the system.
- `Management` service manages the `sessions` and `engines`, coordinates the processes and send callback on events.

The simpliest way is start the services with `docker-compose`. Use the following `docker-compose.yml`:

> You must make sure the networking system is online first!

```yaml
version: "3.8"

services:
    manager:
        restart: always
        image: "362085406009.dkr.ecr.us-east-2.amazonaws.com/liftup-edge-manager:latest"
        volumes:
            - ./configs:/configs
        environment:
            BLUEEDGE_CFG: "/configs/local.yml"
            LOGS_DIR: "/logs/blueedge-manager"
    restful:
        restart: always
        image: "362085406009.dkr.ecr.us-east-2.amazonaws.com/liftup-edge-restful:latest"
        ports:
            - "5000:80"
        volumes:
            - ./configs:/configs
        environment:
            BLUEEDGE_CFG: "/configs/local.yml"
            LOGS_DIR: "/logs/blueedge-manager"
            BLUEEDGE_DEBUG_MODE: 1
```

In the directory that contains the above `docker-compose.yml`, create the configs file that will be mounted to the services:

```bash
mkdir -p configs
nano local.yml
```

Paste the following data:

```yaml
# configs/local.yml

SYSTEM:
  ENVIRONMENT: 'local'
  MONGODB_DB_NAME: 'blueeye_db_dev'
  REDIS_ADDRESS: '192.168.80.186'
  REDIS_PORT: 6380
  KAFKA_BROKERS: ['192.168.80.186:9092']
  MONGODB_CONNECTION_STR: 'mongodb://192.168.80.25:27017'
  MANAGER_BROADCAST_CHANNEL: 'MNG_JZp0px0DalZ7je5kKYlZgddd'
  ENGINE_ATTENDANCE_CHANNEL: 'ATTENDANCE_gPKhENDvX5Zit-SAD3pmXddd'
  SESSION_MANAGE_REQUEST_CHANNEL: 'MANAGE-q-PAH6U_NwE1Uej-BljeSddd'
  USAGE_MANAGE_REQUEST_CHANNEL: 'USAGE-q-PAH6U_NwE1Uej-BljeSddd'
  ENGINE_STATUS_UPDATE_CHANNEL: 'STATUS-gGY7Ot-TAc8aAtfwypz3Fddd'
  TRACING:
    ENABLE: False
```

The config file content must match the engines' config files. See [get started](tutorials/get_started.md).

## On cloud

TODO: Doc for k8s deployment.
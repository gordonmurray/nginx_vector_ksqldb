# Nginx access logs, Vector.dev and kSQLDB

Trying out nginx access logs, [vector.dev](https://vector.dev/) and [kSQLDB](https://ksqldb.io/) for a little bit of stream data fun.

My aim here was to see if vector.dev could read in nginx access logs, push them to a Kafka cluster and for kSQLDB to be query those logs and create a materialized view ( such as a count of visitors ) from the stream.

I created an ec2 instance with ubuntu 22.04. I initially started with a t3.micro but it was too small for the containers to start fully. After scaling up to a t3.xlarge, the containers started successfully.

I installed docker on the machine first taking steps from the Docker site at https://docs.docker.com/engine/install/ubuntu/, then the docker-compose file used to create Zookeeper, Kafka, ksqlDB server and API came from the quickstart on the kSQLdb site at https://ksqldb.io/quickstart.html

The docker compose used is:

```
---
version: '2'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  broker:
    image: confluentinc/cp-kafka:7.3.0
    hostname: broker
    container_name: broker
    depends_on:
      - zookeeper
    ports:
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  ksqldb-server:
    image: confluentinc/ksqldb-server:0.28.2
    hostname: ksqldb-server
    container_name: ksqldb-server
    depends_on:
      - broker
    ports:
      - "8088:8088"
    environment:
      KSQL_LISTENERS: http://0.0.0.0:8088
      KSQL_BOOTSTRAP_SERVERS: broker:9092
      KSQL_KSQL_LOGGING_PROCESSING_STREAM_AUTO_CREATE: "true"
      KSQL_KSQL_LOGGING_PROCESSING_TOPIC_AUTO_CREATE: "true"

  ksqldb-cli:
    image: confluentinc/ksqldb-cli:0.28.2
    container_name: ksqldb-cli
    depends_on:
      - broker
      - ksqldb-server
    entrypoint: /bin/sh
    tty: true
```

Once the containers were running and after trying some of the sample queries on the quickstart guide, I installed nginx-core so that I could generate some logs.

> sudo apt-get install nginx-core

Once nginx was installed, I confirmed access logs were available by default at `/var/logs/nginx/`access.logs

Next I installed vector.dev

> curl --proto '=https' --tlsv1.2 -sSf https://sh.vector.dev | bash

I created the following config file with a File source and a Kafka sink. A source reads from the nginx logs and the sink points to the kafka broker container.

I start vector using

> vector -c vector.toml

The config file content is below. The kafka endpoint is the container started using the docker compose above.

```
[sources.my_source_id]
type = "file"
ignore_older_secs = 600
include = [ "/var/log/nginx/access.log" ]
read_from = "beginning"


[sinks.my_sink_id]
type = "kafka"
inputs = [ "my_source_id" ]
bootstrap_servers = "0.0.0.0:29092"
key_field = "nginx"
topic = "nginx-logs"
compression = "none"

  [sinks.my_sink_id.encoding]
  codec = "json"
```

To check that the kafka container was receiving data, I downloaded kafka locally on the ec2 instance and used its commands to list topics and pull some data

```
wget https://dlcdn.apache.org/kafka/3.1.2/kafka_2.13-3.1.2.tgz
tar -xzf kafka_2.13-3.1.2.tgz
```

```
# list topis
./kafka-topics.sh --list --bootstrap-server broker:9092
```

If the above link doesn't work, there are other kafka releases available here [https://dlcdn.apache.org/kafka/](https://dlcdn.apache.org/kafka/)

I could see the nginx-logs topic created by vector.dev. I queried its data to see some logs using:

```
./kafka-console-consumer.sh --bootstrap-server 0.0.0.0:29092 --from-beginning --topic nginx-logs
```

The next step was to try and create a stream from the kafka topic, using:

```
CREATE STREAM logs (file VARCHAR, host VARCHAR, message VARCHAR, source_type VARCHAR, timestamp VARCHAR)
  WITH (kafka_topic='nginx-logs', value_format='json', partitions=1);
```

I could then show the stream and query it using:

```
show streams;
select * from logs;
```

![query ksqldb stream](https://github.com/gordonmurray/nginx_vector_ksqldb/blob/main/files/ksqldb_select_stream_logs.png?raw=true)

To try out the materialized view I created a visitors table:

```
create table visitors AS select host, count(host) as visits from logs group by host emit changes;
```

And query the table using

```
select * from visitors;
```

![query ksqldb materialized view](https://github.com/gordonmurray/nginx_vector_ksqldb/blob/main/files/ksqldb_select_materialized_view.png?raw=true)

I can use Curl to query the ksqldb server taking examples from the API guide at https://docs.ksqldb.io/en/latest/developer-guide/api/


```
curl -X "POST" "http://localhost:8088/ksql" \
    -H "Accept: application/vnd.ksql.v1+json" \
    -d $'{
        "ksql": "show tables;",
        "streamsProperties": {}
    }'
```

I can query my materialized view from earlier using:

```
curl -X "POST" "http://localhost:8088/query" \
     -H "Accept: application/vnd.ksql.v1+json" \
     -d $'{
  "ksql": "SELECT * FROM visitors;",
  "streamsProperties": {}
}'
```

I get a response like the one below, showing 16 visitors, sam as quergin the view above

```
[
  {
    "header": {
      "queryId": "query_1676410816563",
      "schema": "`HOST` STRING KEY, `VISITS` BIGINT"
    }
  },
  {
    "row": {
      "columns": [
        "ip-100-10-17-179",
        16
      ]
    }
  }
]
```



While this is a quick and basic example, I really like the results.  Wether it is Vector or something like Debezium populating topics on Kafka, kSQLDB can query the data readily using SQL, I can create a marerialized view that updates as new information comes in and I can use Curl to get this information too.

Theres plenty more to learn, like how long can kafka hold the data, how to recover if there is a break in the data and in general handle a much larger volume of data.
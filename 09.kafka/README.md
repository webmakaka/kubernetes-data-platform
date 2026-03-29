## [Kubernetes Data Platform] Install Kafka

### Install Strimzi Operator

```bash
helm repo add strimzi https://strimzi.io/charts/
helm install kafka-operator strimzi/strimzi-kafka-operator --namespace=kafka --create-namespace --debug --version 0.44.0
```

<br/>

### Create Kafka Cluster

```bash
kubectl apply -f deployment/kafka/kafka-cluster.yaml
```

### Create Kafka topic

```bash
kubectl apply -f deployment/kafka/connect-configs-topic.yaml
kubectl apply -f deployment/kafka/connect-offsets-topic.yaml
kubectl apply -f deployment/kafka/connect-status-topic.yaml
kubectl apply -f deployment/kafka/my-topic-topic.yaml
kubectl -n kafka exec -it my-kafka-cluster-kafka-0 bash
[kafka@my-kafka-cluster-kafka-0 kafka]$ bin/kafka-topics.sh --list --bootstrap-server localhost:9092
# connect-configs
# connect-offsets
# connect-status
# my-topic
[kafka@my-kafka-cluster-kafka-0 kafka]$ bin/kafka-topics.sh --list --bootstrap-server 172.25.0.2:32100,172.25.0.3:32100,172.25.0.4:32100
# 172.25.0.2,172.25.0.3,172.25.0.4 is node ip
# connect-configs
# connect-offsets
# connect-status
# my-topic
```

### Produce message to my-topic

```bash
python3 --version
# Python 3.10.12
pip install -r src/requirements.txt
python3 src/producer.py
bin/kafka-console-consumer.sh --topic my-topic --from-beginning --bootstrap-server 172.25.0.2:32100,172.25.0.3:32100,172.25.0.4:32100
```

### Build kafka connect image

```bash
# Replace viet1846 by your repository
docker build -t viet1846/kafka-connect:0.41.0 -f deployment/kafka/Dockerfile deployment/kafka
docker push viet1846/kafka-connect:0.41.0
```

### Create kafka connect and Kafka connector

```bash
kubectl apply -f deployment/kafka/connect.yaml
kubectl apply -f deployment/kafka/sink-my-topic-to-s3-connector.yaml
```

### Debeziums mysql test

```bash
# start mysql source
kubectl apply -f deployment/kafka/namespace.yaml
kubectl apply -f deployment/kafka/deployment.yaml
kubectl apply -f deployment/kafka/serrvice.yaml

# mysql cdc source connector
kubectl apply -f deployment/kafka/debezium-connector-mysql.yaml
# update data source
kubectl run mysql-client --image=mysql:5.7 -it --rm --restart=Never --namespace=kafka -- /bin/bash
bash-4.2# mysql -uroot -P3306 -hmysql.data-source -p
mysql> use inventory;
mysql> update customers set first_name="Sally Marie" where id=1001;
# Query OK, 1 row affected (0.27 sec)
# Rows matched: 1  Changed: 1  Warnings: 0

#check
kubectl -n kafka exec -it my-kafka-cluster-kafka-0 bash
[kafka@my-kafka-cluster-kafka-0 kafka]$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mysql.inventory.customers --from-beginning
#{"before":null,"after":{"id":1001,"first_name":"Sally Marie 1","last_name":"Thomas","email":"sally.thomas@acme.com"},"source":{"version":"2.4.2.Final","connector":"mysql","name":"mysql","ts_ms":1717999936000,"snapshot":"first_in_data_collection","db":"inventory","sequence":null,"table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":953,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1717999936070,"transaction":null}
#{"before":null,"after":{"id":1002,"first_name":"George","last_name":"Bailey","email":"gbailey@foobar.com"},"source":{"version":"2.4.2.Final","connector":"mysql","name":"mysql","ts_ms":1717999936000,"snapshot":"true","db":"inventory","sequence":null,"table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":953,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1717999936070,"transaction":null}
#{"before":null,"after":{"id":1003,"first_name":"Edward","last_name":"Walker","email":"ed@walker.com"},"source":{"version":"2.4.2.Final","connector":"mysql","name":"mysql","ts_ms":1717999936000,"snapshot":"true","db":"inventory","sequence":null,"table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":953,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1717999936070,"transaction":null}
#{"before":null,"after":{"id":1004,"first_name":"Anne","last_name":"Kretchmar","email":"annek@noanswer.org"},"source":{"version":"2.4.2.Final","connector":"mysql","name":"mysql","ts_ms":1717999936000,"snapshot":"last_in_data_collection","db":"inventory","sequence":null,"table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":953,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1717999936070,"transaction":null}
#{"before":{"id":1001,"first_name":"Sally Marie 1","last_name":"Thomas","email":"sally.thomas@acme.com"},"after":{"id":1001,"first_name":"Sally Marie","last_name":"Thomas","email":"sally.thomas@acme.com"},"source":{"version":"2.4.2.Final","connector":"mysql","name":"mysql","ts_ms":1718001145000,"snapshot":"false","db":"inventory","sequence":null,"table":"customers","server_id":223344,"gtid":null,"file":"mysql-bin.000003","pos":1197,"row":0,"thread":34,"query":null},"op":"u","ts_ms":1718001145612,"transaction":null}

# Sink to S3 (Minio)
kubectl apply -f deployment/kafka/sink-mysql-kafka-topic-to-s3-connector.yaml
```

### Debeziums postgres test

```bash
# Create postgres DB
kubectl apply -f deployment/postgres/postgres.yaml

# Create debezium cdc source connector
kubectl apply -f deployment/kafka/debezium-connector-postgres.yaml
kubectl apply -f deployment/postgres/postgresql-client.yml
kubectl -n kafka exec -it postgresql-client sh
data_engineer=# \dt inventory.*
#                   List of relations
#   Schema   |       Name       | Type  |     Owner
# -----------+------------------+-------+---------------
#  inventory | customers        | table | data_engineer
#  inventory | geom             | table | data_engineer
#  inventory | orders           | table | data_engineer
#  inventory | products         | table | data_engineer
#  inventory | products_on_hand | table | data_engineer
#  inventory | spatial_ref_sys  | table | data_engineer
# (6 rows)
update inventory.customers set first_name='Sally Marie' where id=1001;

#check
kubectl -n kafka exec -it my-kafka-cluster-kafka-0 bash
[kafka@my-kafka-cluster-kafka-0 kafka]$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic postgres.inventory.customers --from-beginning

# Sink to s3 (Minio)
kubectl apply  -f deployment/kafka/sink-postgres-kafka-topic-to-s3-connector.yaml
```

### Query data by Trino

```bash
trino> CREATE SCHEMA lakehouse.raw WITH (location = 's3a://lakehouse/raw/');

trino> CREATE TABLE IF NOT EXISTS minio.raw.mytopic(
	DOLocationID            bigint,
	RatecodeID              bigint,
	fare_amount             double,
	tpep_dropoff_datetime   varchar,
	congestion_surcharge    double,
	VendorID                bigint,
	passenger_count         bigint,
	tolls_amount            bigint,
	Airport_fee             bigint,
	improvement_surcharge   bigint,
	messageTS               varchar,
	trip_distance           double,
	store_and_fwd_flag      varchar,
	payment_type            bigint,
	total_amount            double,
	extra                   bigint,
	tip_amount              double,
	mta_tax                 double,
	tpep_pickup_datetime    varchar,
	PULocationID            bigint,
    year                    varchar,
    month                   varchar,
    day                     varchar,
    hour                    varchar
)WITH
(
 	format = 'JSON',
 	partitioned_by = ARRAY[ 'year', 'month', 'day', 'hour' ],
 	external_location = 's3a://kafka/topics/my-topic'
);
trino> CALL minio.system.sync_partition_metadata('raw', 'mytopic', 'FULL');
trino> select * from minio.raw.mytopic;
trino> CREATE TABLE IF NOT EXISTS minio.raw.mytopic_textfile(
	json_string             varchar,
    year                    varchar,
    month                   varchar,
    day                     varchar,
    hour                    varchar
)WITH
(
 	format = 'TEXTFILE',
 	partitioned_by = ARRAY[ 'year', 'month', 'day', 'hour' ],
 	external_location = 's3a://kafka/topics/my-topic'
);
trino> CALL minio.system.sync_partition_metadata('raw', 'mytopic_textfile', 'FULL');
trino> select * from minio.raw.mytopic_textfile;


CREATE TABLE IF NOT EXISTS minio.raw.mysql_customers(
	json_string             varchar,
    year                    varchar,
    month                   varchar,
    day                     varchar,
    hour                    varchar
)WITH
(
 	format = 'TEXTFILE',
 	partitioned_by = ARRAY[ 'year', 'month', 'day', 'hour' ],
 	external_location = 's3a://kafka/topics/mysql.inventory.customers'
);
trino> CALL minio.system.sync_partition_metadata('raw', 'mysql_customers', 'FULL');
trino> select * from minio.raw.mysql_customers;
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

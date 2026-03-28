## [Kubernetes Data Platform] Install Trino

<br/>

**Should be installed already:**

- minio
- hive metastore

<br/>

```bash
$ helm repo add trino https://trinodb.github.io/charts
$ helm upgrade --install trino -n trino -f deployment/trino/trino-values.yaml trino/trino --create-namespace --debug --version 0.21.0
```

<br/>

**Test**

```bash
$ kubectl -n trino get po
$ kubectl -n trino exec -it trino-coordinator-bd4568cd8-jn6sc trino

trino> CREATE SCHEMA lakehouse.raw WITH (location = 's3a://lakehouse/raw/');
trino> CREATE TABLE IF NOT EXISTS minio.raw.yellow_tripdata (
 vendor_name varchar,
 Trip_Pickup_DateTime varchar,
 Trip_Dropoff_DateTime varchar,
 Passenger_Count bigint,
 Trip_Distance double,
 Start_Lon double,
 Start_Lat double,
 Rate_Code double,
 store_and_forward double,
 End_Lon double,
 End_Lat double,
 Payment_Type Varchar,
 Fare_Amt double,
 surcharge double,
 mta_tax double,
 Tip_Amt double,
 Tolls_Amt double,
 Total_Amt double,
 y varchar,
 m varchar
)
WITH
(
    format = 'PARQUET',
    external_location = 's3a://lakehouse/raw/yellow_tripdata',
    partitioned_by = ARRAY[ 'y', 'm' ]
);

trino> CALL minio.system.sync_partition_metadata('raw', 'yellow_tripdata', 'FULL');

trino> CREATE TABLE IF NOT EXISTS lakehouse.raw.yellow_tripdata_trino_iceberg (
 vendor_name varchar,
 Trip_Pickup_DateTime varchar,
 Trip_Dropoff_DateTime varchar,
 Passenger_Count bigint,
 Trip_Distance double,
 Start_Lon double,
 Start_Lat double,
 Rate_Code double,
 store_and_forward double,
 End_Lon double,
 End_Lat double,
 Payment_Type Varchar,
 Fare_Amt double,
 surcharge double,
 mta_tax double,
 Tip_Amt double,
 Tolls_Amt double,
 Total_Amt double,
 y varchar,
 m varchar
)
WITH
(
    format = 'PARQUET',
    format_version = 2,
    partitioning = ARRAY[ 'y', 'm' ]
);

trino> insert into lakehouse.raw.yellow_tripdata_trino_iceberg
select * from lakehouse.raw.yellow_tripdata;
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

## [Kubernetes Data Platform] Install Trino

```bash
$ helm repo add minio https://charts.min.io/
$ helm repo update
```

<br/>

```
# helm uninstall minio -n minio
$ helm upgrade --install minio -n minio -f deployment/minio/minio-values.yaml minio/minio --create-namespace --debug --version 5.4.0
$ kubectl -n minio get po

$ kubectl get no -owide
NAME              STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION     CONTAINER-RUNTIME
marley-minikube   Ready    control-plane   19m   v1.35.0   192.168.49.2   <none>        Debian GNU/Linux 12 (bookworm)   6.5.0-45-generic   docker://29.2.0
```

<br/>

```bash
$ sudo vi /etc/hosts
192.168.49.2 minio.lakehouse.local
192.168.49.2 console.minio.lakehouse.local
```

**Test**

```bash
# init data test
$ kubectl run minio-client --image=minio/minio -it --rm --restart=Never --command /bin/bash
# mc alias set minio http://minio.minio.svc.cluster.local:9000 admin password
# curl https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2009-01.parquet -o yellow_tripdata_2009-01.parquet
# mc cp ./yellow_tripdata_2009-01.parquet minio/lakehouse/raw/yellow_tripdata/y=2009/m=01/yellow_tripdata_2009-01.parquet
```

<br/>

## Install Hive Metastore

### hive-metastore-postgresql

https://artifacthub.io/packages/helm/bitnami/postgresql

```bash
$ helm upgrade --install metastore-db -n metastore -f deployment/hive/hive-metastore-postgres-values.yaml helm-chart/postgresql-18.5.14/postgresql --create-namespace --debug --version 15.4.2
```

### Hive metastore

```bash
$ helm upgrade --install hive-metastore -n metastore -f deployment/hive/hive-metastore-values.yaml ../charts/hive-metastore --create-namespace --debug
```

## Install Trino

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

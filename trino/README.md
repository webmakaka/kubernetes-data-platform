## Note
```text
source: https://github.com/IvanWoo/trino-on-kubernetes
```

## Create k8s cluster (kind)
```bash
kind create cluster --name dev --config deployment/kind/kind-config.yaml 
```
## Nginx ingress
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --set controller.hostNetwork=true,controller.service.type="",controller.kind=DaemonSet --namespace ingress-nginx --version 4.10.1 --create-namespace --debug

k -n ingress-nginx get po -owide
```

## Install Mysql Data Source
```bash
k apply -f deployment/kafka/namespace.yaml
k apply -f deployment/kafka/deployment.yaml
k apply -f deployment/kafka/serrvice.yaml
```

## Install PostgreSQL Data Source
```bash
k apply -f deployment/kafka/debezium-connector-postgres.yaml
k apply -f deployment/postgres/postgresql-client.yml
k -n kafka exec -it postgresql-client sh
data_engineer=# \dt inventory.*
```

## Install Minio on Kubernetes

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

## Destroy kind

```bash
kind delete cluster -n dev 
```


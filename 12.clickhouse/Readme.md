## [Kubernetes Data Platform] Install Clickhouse

<br/>

### Altinity

https://github.com/altinity/clickhouse-operator

https://docs.altinity.com/altinitykubernetesoperator/quickstartinstallation/

<br/>

```bash

$ helm repo add altinity https://helm.altinity.com
$ helm repo update

$ helm upgrade --install clickhouse-operator \
  altinity/altinity-clickhouse-operator \
  --version 0.26.2 \
  --namespace clickhouse \
  --create-namespace
```

<br/>

```
$ helm list --output yaml -n clickhouse | grep app_version
- app_version: 0.26.2
```

<br/>

```
$ kubectl get deployment.apps -n clickhouse
NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
clickhouse-operator-altinity-clickhouse-operator   1/1     1            1           57s
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: "clickhouse.altinity.com/v1"
kind: "ClickHouseInstallation"
metadata:
  name: cluster01
  namespace: clickhouse
spec:
  templates:
    podTemplates:
      - name: clickhouse-pod-template
        spec:
          containers:
            - name: clickhouse
              image: altinity/clickhouse-server:25.8.16.10001.altinitystable
  configuration:
    clusters:
      - name: cluster01
        layout:
          shardsCount: 1
          replicasCount: 1
        templates:
          podTemplate: clickhouse-pod-template
EOF
```

<br/>

```bash
$ kubectl get clickhouseinstallation -n clickhouse
NAME        STATUS       CLUSTERS   HOSTS   HOSTS-COMPLETED   AGE   SUSPEND
cluster01   InProgress   1          1                         52s
```

<br/>

### Connecting to your cluster with kubectl exec

<br/>

```bash
$ kubectl get pods -n clickhouse | grep cluster01
chi-cluster01-cluster01-0-0-0                                     1/1     Running   0          82s
```

<br/>

```bash
$ kubectl exec -it chi-cluster01-cluster01-0-0-0 -n clickhouse -- clickhouse-client
```

<br/>

```sql
DESCRIBE url(
  'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_0.gz',
  'TabSeparatedWithNames'
);
```

<br/>

```sql
SELECT
    passenger_count,
    avg(toFloat32(total_amount))
FROM url(
    'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_0.gz',
    'TabSeparatedWithNames'
)
GROUP BY passenger_count
ORDER BY passenger_count ASC;
```

<br/>

```sql
CREATE TABLE trips (
    trip_id             UInt32,
    pickup_datetime     DateTime,
    dropoff_datetime    DateTime,
    pickup_longitude    Nullable(Float64),
    pickup_latitude     Nullable(Float64),
    dropoff_longitude   Nullable(Float64),
    dropoff_latitude    Nullable(Float64),
    passenger_count     UInt8,
    trip_distance       Float32,
    fare_amount         Float32,
    extra               Float32,
    tip_amount          Float32,
    tolls_amount        Float32,
    total_amount        Float32,
    payment_type        Enum('CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4, 'UNK' = 5),
    pickup_ntaname      LowCardinality(String),
    dropoff_ntaname     LowCardinality(String)
)
ENGINE = MergeTree
PRIMARY KEY (pickup_datetime, dropoff_datetime);
```

<br/>

```sql
INSERT INTO trips
SELECT
    trip_id,
    pickup_datetime,
    dropoff_datetime,
    pickup_longitude,
    pickup_latitude,
    dropoff_longitude,
    dropoff_latitude,
    passenger_count,
    trip_distance,
    fare_amount,
    extra,
    tip_amount,
    tolls_amount,
    total_amount,
    payment_type,
    pickup_ntaname,
    dropoff_ntaname
FROM s3(
    'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_{0..2}.gz',
    'TabSeparatedWithNames'
);
```

<br/>

```sql
SELECT count() from trips;
```

<br/>

```sql
SELECT
    passenger_count,
    avg(toFloat32(total_amount))
FROM trips
GROUP BY passenger_count
ORDER BY passenger_count ASC;
```

<br/>

### ClickHouse ing

https://github.com/ClickHouse/clickhouse-operator

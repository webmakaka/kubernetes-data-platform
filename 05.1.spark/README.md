## [Kubernetes Data Platform] Install Spark Operator

<br/>

```bash
$ helm repo add spark-operator https://kubeflow.github.io/spark-operator

$ helm upgrade \
    --install spark-operator spark-operator/spark-operator \
    --namespace spark-operator \
    --create-namespace \
    --set webhook.enable=true \
    --set image.tag=v1beta2-1.4.6-3.5.0 \
    --debug \
    --version 1.3.2

$ kubectl create role spark-operator-submitter --verb=create,get --resource=sparkapplications,pods/log --namespace=spark-operator

$ kubectl create rolebinding airflow-worker-spark-submitter --role=spark-operator-submitter --serviceaccount=airflow:airflow-worker --namespace=spark-operator
```

<br/>

### Test spark pi

```bash
$ kubectl apply -f jobs/spark_pi.yaml
$ kubectl -n spark-operator logs -f pyspark-pi-driver
```

<br/>

### Build Base images

```bash
$ docker build -t viet1846/spark-iceberg:3.4.1 -f Dockerfile.spark-iceberg .
```

<br/>

### Build application images

```bash
$ docker build -t viet1846/spark-lakehouse:v1 -f Dockerfile .
```

<br/>

### Submit job

```bash
$ kubectl apply -f jobs/main_iceberg.yaml
$ kubectl -n spark-operator get pods

# NAME                                READY   STATUS      RESTARTS   AGE
# pyspark-pi-driver                   0/1     Completed   0          4m
# spark-iceberg-driver                0/1     Completed   0          52m
# spark-operator-7f88f7dbf-46k6c      1/1     Running     0          120m
# spark-operator-webhook-init-p796r   0/1     Completed   0          120m

$ kubectl -n spark-operator logs -f spark-iceberg-driver
```

<br/>

## Using Trino query table create by Spark

```bash
$ kubectl -n trino exec -it trino-coordinator-b597bcd8c-f6vf7 trino

trino> select * from lakehouse.raw.taxis_spark limit 10;
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

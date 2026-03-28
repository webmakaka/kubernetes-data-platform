## [Kubernetes Data Platform] Install Airflow

<br/>

```bash
// docker build -t viet1846/airflow:2.8.3 .
// docker push viet1846/airflow:2.8.3
```

<br/>

```bash
// $ helm repo add airflow https://airflow.apache.org/
// helm upgrade --install airflow airflow/airflow -f deployment/airflow/airflow-values.yaml --namespace airflow --create-namespace --debug --version 1.13.1 --timeout 600s
```

<br/>

```bash
// $ helm uninstall airflow --namespace airflow

$ helm upgrade --install airflow -f deployment/airflow/airflow-values.yaml --namespace airflow --create-namespace --debug --timeout 600s ../helm-charts/airflow
```

<br/>

```bash
# kubectl create secret generic airflow-ssh-secret --from-file=gitSshKey=/path/to/.ssh/airflowsshkey -n airflow
```

<br/>

```bash
$ sudo vi /etc/hosts
192.168.49.2 airflow.lakehouse.local
```

<br/>

```
// OK!
// admin / admin
http://airflow.lakehouse.local
```

<br/>

### Config S3 Connection and Kubernetes Connection in Airflow UI

<br/>

```yaml
Connection Id: s3_default
Connection Type: Amazon Web Services
AWS Access Key ID: admin
AWS Secret Access Key: password
Extra: { 'endpoint_url': 'http://minio.minio.svc.cluster.local:9000' }
```

<br/>

```yaml
Connection Id: kubernetes_default
Connection Type: Kubernetes Cluster Connection
In cluster configuration: yes
Disable SSL: yes
```

<br/>

### Config Airflow permission to submit Spark job

```bash
$ kubectl create role spark-operator-submitter --verb=create,get --resource=sparkapplications,pods/log --namespace=spark-operator

$ kubectl create rolebinding airflow-worker-spark-submitter --role=spark-operator-submitter --serviceaccount=airflow:airflow-worker --namespace=spark-operator
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

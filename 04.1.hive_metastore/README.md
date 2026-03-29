## [Kubernetes Data Platform] Install Hive Metastore

<br/>

### Postgresql for Hive metastore

https://artifacthub.io/packages/helm/bitnami/postgresql

<br/>

```bash
$ helm upgrade --install metastore-db -n metastore -f deployment/hive/hive-metastore-postgres-values.yaml ../helm-charts/postgresql-18.5.14/postgresql --create-namespace --debug
```

<br/>

### Hive metastore

```bash
$ helm upgrade --install hive-metastore -n metastore -f deployment/hive/hive-metastore-values.yaml ../helm-charts/hive-metastore --create-namespace --debug
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

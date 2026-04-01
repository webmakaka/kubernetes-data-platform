## [Kubernetes Data Platform] Install Postgres

<br/>

### Bitnami

```bash
$ helm upgrade --install postgres -n postgres --create-namespace -f deployment/postgres/postgres-values.yaml ../helm-charts/postgresql/postgresql-18.5.14/postgresql --debug
```

<br/>

### [Test] CloudNativePG Operator Helm Chart

<br/>

https://artifacthub.io/packages/helm/cloudnative-pg/cloudnative-pg

<br/>

```
$ helm repo add cloudnative-pg https://cloudnative-pg.io/charts/
$ helm repo update

$ helm search repo cloudnative-pg

$ cd Kubernetes-Data-Platform/helm-charts/operator/

$ helm pull cloudnative-pg/cloudnative-pg \
  --version 0.28.0 \
  --untar \
  --untardir cloudnative-pg-0.28.0


$ helm install my-cnpg ../helm-charts/postgresql/operator/cloudnative-pg-0.28.0/cloudnative-pg/ \
  --namespace cnpg-system \
  --create-namespace
```

<br/>

```
$ k get pods -n cnpg-system
NAME                                     READY   STATUS    RESTARTS   AGE
my-cnpg-cloudnative-pg-dc84469cb-g42bp   1/1     Running   0          33s
```

<br/>

```
$ kubectl create namespace postgres-database
```

<br/>

```
$ kubectl create secret generic postgres-db-secret \
  --namespace postgres-database \
  --from-literal=username=postgres_user \
  --from-literal=password=my_strong_password
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres
  namespace: postgres-database
spec:
  instances: 1
  # imageName: ghcr.io/cloudnative-pg/postgresql:18
  bootstrap:
    initdb:
      database: postgres_db
      owner: postgres_user
      secret:
        name: postgres-db-secret
  storage:
    size: 5Gi
    storageClass: standard
EOF
```

<br/>

```
$ kubectl exec -it svc/postgres-rw -n postgres-database -- psql -U postgres -c "SELECT version();"
PostgreSQL 18.3 (Debian 18.3-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64
```

<br/>

```
// OK!
// password: my_strong_password
$ kubectl exec -it postgres-1 -n postgres-database -- psql -h localhost -U postgres_user -d postgres_db
```

<br/>

```
$ kubectl port-forward svc/postgres-rw -n postgres-database 5432:5432
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

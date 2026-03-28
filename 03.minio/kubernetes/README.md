## [Kubernetes Data Platform] Install Minio

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

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

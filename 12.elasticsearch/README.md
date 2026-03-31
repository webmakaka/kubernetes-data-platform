## [Kubernetes Data Platform] Install Elasticsearch

<br/>

https://artifacthub.io/packages/helm/elastic/eck-operator/2.12.1

<br/>

```
// $ helm repo add elastic https://helm.elastic.co
```

<br/>

```
// Do not works for me, because Russia has been banned
// $ helm install elastic-operator elastic/eck-operator -n elastic --create-namespace --version 2.12.1
```

<br/>

```
$ helm install elastic-operator ../helm-charts/eck-operator-2.12.1/eck-operator -n elastic --create-namespace
```

<br/>

```
$ kubectl apply -f deployment/elastic_cluster.yaml -n elastic
$ kubectl apply -f deployment/kibana.yaml -n elastic
```

<br/>

```
$ kubectl get pods -n elastic
NAME                        READY   STATUS    RESTARTS   AGE
elastic-es-default-0        1/1     Running   0          39s
elastic-operator-0          1/1     Running   0          14m
kibana-kb-54c4648c8-6cgv6   1/1     Running   0          30s
```

<br/>

```
$ kubectl get elastic -n elastic
NAME                                                 HEALTH   NODES   VERSION   PHASE   AGE
elasticsearch.elasticsearch.k8s.elastic.co/elastic   green    1       8.13.0    Ready   60s

NAME                                  HEALTH   NODES   VERSION   AGE
kibana.kibana.k8s.elastic.co/kibana   green    1       8.13.0    53s
```

<br/>

```
$ kubectl get elasticsearch -n elastic
NAME      HEALTH   NODES   VERSION   PHASE   AGE
elastic   green    1       8.13.0    Ready   77s
```

<br/>

```
$ kubectl describe elastic -n elastic
```

<br/>

```
$ kubectl get pods -n elastic
NAME                        READY   STATUS    RESTARTS   AGE
elastic-es-default-0        1/1     Running   0          100m
elastic-operator-0          1/1     Running   0          102m
kibana-kb-87b948855-4rfdg   1/1     Running   0          100m
```

<br/>

```
$ kubectl get svc -n elastic
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
elastic-es-default         ClusterIP   None             <none>        9200/TCP   101m
elastic-es-http            ClusterIP   10.110.241.94    <none>        9200/TCP   101m
elastic-es-internal-http   ClusterIP   10.104.35.233    <none>        9200/TCP   101m
elastic-es-transport       ClusterIP   None             <none>        9300/TCP   101m
elastic-operator-webhook   ClusterIP   10.105.151.128   <none>        443/TCP    103m
kibana-kb-http             ClusterIP   10.107.181.144   <none>        5601/TCP   101m
```

<br/>

```
$ kubectl port-forward svc/kibana-kb-http 5601:5601 -n elastic
```

<br/>

```
$ kubectl get secret elastic-es-elastic-user -n elastic -o go-template='{{.data.elastic | base64decode}}'
```

<br/>

```
// Kibana will not accept regular HTTP protocol connections
// elastic /
https://localhost:5601
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

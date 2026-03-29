## [Kubernetes Data Platform] Install Postgres

<br/>

```bash
$ helm upgrade --install postgres -n postgres --create-namespace -f deployment/postgres/postgres-values.yaml.yaml ../helm-charts/postgresql-18.5.14/postgresql  --debug
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

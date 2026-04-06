## [Kubernetes Data Platform] Install jupyterhub

<br/>

```bash
$ helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
$ helm repo update
```

<br/>

```yaml
$ cat > jupyterhub-values.yaml <<EOF
singleuser:
  image:
    name: jupyter/scipy-notebook # Образ с JupyterLab
    tag: latest
  defaultUrl: "/lab" # Включает JupyterLab по умолчанию вместо классического Notebook
  storage:
    dynamic:
      storageClass: standard # Класс для автоматического создания дисков (PVC)
  memory:
    limit: 2G
    guarantee: 1G
EOF
```

<br/>

```bash
$ helm upgrade --cleanup-on-fail \
  --install jupyterhub jupyterhub/jupyterhub \
  --namespace jupyterhub --create-namespace \
  --values jupyterhub-values.yaml
```

<br/>

```bash
$ kubectl port-forward svc/proxy-public 8000:80 -n jupyterhub
```

```
// OK!
// root /
http://localhost:8000/hub/login?next=%2Fhub%2F
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

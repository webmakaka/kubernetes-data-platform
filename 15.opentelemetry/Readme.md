## [Kubernetes Data Platform] Install OpenTelemetry

<br/>

```
https://github.com/open-telemetry/opentelemetry-operator
```

<br/>

https://www.youtube.com/watch?v=PrIYUO51rMU

<br/>

https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/monitoring/opentelemetry/kubernetes

<br/>

### OpenTelemetry Operator chart

<br/>

```shell
$ helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
$ helm repo update

$ helm search repo open-telemetry --versions
```

</br>

```shell
$ OTEL_VERSION=0.93.1
```

<br/>

### Cert-manager chart

We will need cert-manager deployed which Otel uses for local TLS certificate management

```shell
$ helm repo add jetstack https://charts.jetstack.io

$ helm search repo jetstack --versions
```

</br>

```shell
$ CERTMANAGER_VERSION=v1.18.2
```

<br/>

### Install helm charts

Install cert-manager:

```shell
$ helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version $CERTMANAGER_VERSION \
    --set crds.enabled=true \
    --set startupapicheck.timeout="5m"
```

<br/>

### Install OpenTelemetry Operator:

```shell
$ helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace opentelemetry-operator-system \
  --create-namespace \
  --version $OTEL_VERSION \
  --values opentelemetry-values.yaml
```

<br/>

```shell
$ kubectl get pods -n cert-manager
$ kubectl get pods -n opentelemetry-operator-system$
```

<br/>

### Create a Collector

<br/>

```bash
$ kubectl create namespace monitoring
```

<br/>

```yaml
$ cat << EOF | kubectl apply -n monitoring -f -
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: trace-collector
spec:
  image: ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector-contrib:0.131.1
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15
      batch:
        send_batch_size: 10000
        timeout: 10s

    exporters:
      # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
      debug: {}
      otlp/tempo:
        endpoint: "http://tempo.grafana.svc.cluster.local:4317"
        tls:
          insecure: true
      prometheusremotewrite:
        endpoint: "http://prometheus-operated:9090/api/v1/write"
        tls:
          insecure: true
          insecure_skip_verify: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [debug, otlp/tempo]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [prometheusremotewrite]
EOF
```

<br/>

```shell
$ kubectl -n monitoring get pods
$ kubectl -n monitoring get svc
```

<br/>

## Tracing

### Instrumentation

<br/>

```yaml
$ cat << EOF | kubectl apply -n monitoring -f -
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: app-instrumentation
spec:
  exporter:
    endpoint: http://trace-collector-collector.monitoring.svc.cluster.local:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: '1'
  dotnet:
    env:
      - name: OTEL_EXPORTER_OTLP_TRACES_PROTOCOL
        value: "grpc"
      - name: OTEL_LOG_LEVEL
        value: "error"
      - name: OTEL_DOTNET_AUTO_LOG_DIRECTORY
        value: /tmp
      - name: "OTEL_DOTNET_AUTO_METRICS_INSTRUMENTATION_ENABLED"
        value: "true"
      - name: "OTEL_DOTNET_AUTO_METRICS_INSTRUMENTATION_ASPNETCORE_ENABLED"
        value: "true"
      - name: "OTEL_DOTNET_AUTO_METRICS_INSTRUMENTATION_HTTPCLIENT_ENABLED"
        value: "true"
  go:
    env:
      - name: OTEL_EXPORTER_OTLP_PROTOCOL
        value: "grpc"
      - name: OTEL_PROPAGATORS
        value: "tracecontext,baggage"
EOF
```

<br/>

### Deploy Microservices

<br/>

https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/monitoring/opentelemetry/applications

<br/>

```shell
$ kubectl apply -f monitoring/opentelemetry/applications/playlists-api/
$ kubectl apply -f monitoring/opentelemetry/applications/playlists-db/
$ kubectl apply -f monitoring/opentelemetry/applications/videos-web/
$ kubectl apply -f monitoring/opentelemetry/applications/videos-db/
$ kubectl apply -f monitoring/opentelemetry/applications/videos-api/
```

<br/>

```shell
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
playlists-api-84d5fbdcc4-llc9p   2/2     Running   0          2m41s
playlists-db-67489585db-49n8f    1/1     Running   0          2m31s
videos-api-865c68989d-b8cb2      1/1     Running   0          2m17s
videos-db-bcc6f8587-4qf62        1/1     Running   0          2m22s
videos-web-7d845fdbf9-tp9r4      1/1     Running   0          2m26s
```

<br/>

**Generate some traffic with `port-forward`**

<br/>

```shell
$ kubectl port-forward svc/videos-web 8080:80
$ kubectl port-forward svc/playlists-api 8081:80
```

<br/>

### Tracing Data Store

In this guide, we'll use a Tempo database for tracing data

<br/>

```shell
$ helm repo add grafana https://grafana.github.io/helm-charts
$ helm repo update

$ helm search repo grafana/tempo

$ TEMPO_VERSION=1.23.3

$ helm install tempo grafana/tempo \
    --create-namespace \
    --namespace grafana \
    --version $TEMPO_VERSION \
    --values tempo-values.yaml
```

<br/>

### Metrics Data Store

In this guide, we'll use a Prometheus for our metrics data

```shell
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm search repo prometheus-community --versions

$ PROMETHEUS_STACK_VERSION=77.5.0

$ helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
 --version ${PROMETHEUS_STACK_VERSION} \
 --namespace prometheus-operator-system \
 --create-namespace \
 --set prometheusOperator.enabled=true \
 --set prometheusOperator.nodeSelector."kubernetes\.io/os"=linux \
 --set prometheusOperator.fullnameOverride="prometheus-operator" \
 --set prometheusOperator.manageCrds=true \
 --set alertmanager.enabled=false \
 --set grafana.enabled=false \
 --set prometheus-node-exporter.enabled=false \
 --set nodeExporter.enabled=false \
 --set kubeStateMetrics.enabled=false \
 --set prometheus.enabled=false
```

<br/>

```shell
$ kubectl -n prometheus-operator-system get pods
```

<br/>

**Deploy our dedicated Prometheus for metrics storage**

<br/>

```yaml
$ cat << EOF | kubectl apply -n monitoring -f -
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  enableRemoteWriteReceiver: true
  enableFeatures:
    - remote-write-receiver
  evaluationInterval: 30s
  imagePullPolicy: IfNotPresent
  nodeSelector:
    kubernetes.io/os: linux
  replicas: 1
  retention: 10d
  scrapeInterval: 30s
EOF
```

<br/>

```shell
$ kubectl -n monitoring get pods
NAME                                         READY   STATUS    RESTARTS   AGE
prometheus-prometheus-0                      2/2     Running   0          5m51s
trace-collector-collector-5849765bd7-nmp6l   1/1     Running   0          25m
```

<br/>

### Dashboards

In this guide I use a simple Grafana for dashboards

```shell
$ helm search repo grafana/grafana

$ GRAFANA_VERSION=9.4.4

$ helm install grafana grafana/grafana \
 --namespace grafana \
 --version $GRAFANA_VERSION \
 --values grafana-values.yaml
```

<br/>

```shell
$ kubectl -n grafana get pods
NAME                       READY   STATUS    RESTARTS   AGE
grafana-5d554f7fb6-9hfrg   1/1     Running   0          119s
tempo-0                    1/1     Running   0          7m23s
```

<br/>

### Access Grafana

We can `port-forward` to Grafana

<br/>

```shell
$ kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

<br/>

```shell
$ kubectl -n grafana port-forward svc/grafana 3000:80
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>

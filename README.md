# test-monitoring-tools

## App (FastApi)

Deploy app from repo [`test-app`](https://github.com/CandelaV/test-app) using https://pypi.org/project/prometheus-fastapi-instrumentator/ to expose `/metrics` endpoint for prometheus.


## Prometheus

### Installation k8s

https://devapo.io/blog/technology/how-to-set-up-prometheus-on-kubernetes-with-helm-charts/

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

```
kubectl create ns monitoring
```

```
helm show values prometheus-community/kube-prometheus-stack > kube-prometheus-stack-values.yaml
```

```
helm -n monitoring install prometheus prometheus-community/kube-prometheus-stack -f kube-prometheus-stack-values.yaml
```

```
kubectl -n monitoring port-forward pod/prometheus-prometheus-kube-prometheus-prometheus-0 9090 
```

```
kubectl -n monitoring port-forward $(kubectl get pod -n monitoring | awk '/prometheus-grafana-/{print $1}') 3000 
```

```
kubectl -n monitoring logs $(kubectl get pod -n monitoring | awk '/prometheus-grafana-/{print $1}') -f
```

```
user: `admin`
pass: `prom-operator`
```

#### discover targets in other namespaces

Update namespace with label

https://github.com/helm/charts/issues/20581#issuecomment-723991976

```
kubectl label namespace test-python-app prometheus=monitoring
kubectl label namespace monitoring prometheus=monitoring
kubectl label namespace kube-node-lease prometheus=monitoring
kubectl label namespace kube-public prometheus=monitoring
kubectl label namespace kube-system prometheus=monitoring
kubectl label namespace default prometheus=monitoring
```

modify values

```
    ruleSelectorNilUsesHelmValues: false
    ruleNamespaceSelector:
      matchLabels:
        prometheus: monitoring
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorNamespaceSelector:
      matchLabels:
        prometheus: monitoring
    podMonitorSelectorNilUsesHelmValues: false
    podMonitorNamespaceSelector:
      matchLabels:
        prometheus: monitoring
    probeSelectorNilUsesHelmValues: false
    probeNamespaceSelector:
      matchLabels:
        prometheus: monitoring
    scrapeConfigSelectorNilUsesHelmValues: false
    scrapeConfigNamespaceSelector:
      matchLabels:
        prometheus: monitoring
```

```
helm -n monitoring upgrade --install prometheus prometheus-community/kube-prometheus-stack -f kube-prometheus-stack-values.yaml
```

#### create service monitor with CRD

https://www.youtube.com/watch?v=6xmWr7p5TE0

Create `service-monitor-test-app.yaml` making sure that:
 - `spec.jobLabel` is correct the correct one from app's service label `job` value
 - `spec.endpoints[0].port` is the correct from app's service port name
 - `spec.selector.matchLabels` is correct given app's service label `app` value
 - `metadata.labels` contains the same label from `kubectl -n prometheus get prometheus.monitoring.coreos.com -o yaml`
    ```
    serviceMonitorSelector:
      matchLabels:
        release: prometheus
    ```


```
kubectl -n test-python-app apply -f service-monitor-test-app.yaml
```

## Grafana

### Installation k8s

Done as part of `prometheus-community/kube-prometheus-stack` helm chart.

### Dashboard



## Loki Grafana

### Installation k8s

``` 
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

```
helm show values grafana/loki > loki-values.yaml
```

```
helm -n monitoring install loki grafana/loki -f loki-values.yaml
```

```
helm -n monitoring upgrade loki grafana/loki -f loki-values.yaml
```

auth connection issue in grafana: https://github.com/grafana/loki/issues/9756

Connect in grafana pointing to `http://loki-gateway`.

To install in monolithic mode https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/. use `loki-monolithic.values.yaml`.


## FluentBit

### Installation k8s

```
helm repo add fluent https://fluent.github.io/helm-charts
```

```
helm search repo fluent
```

```
helm show values fluent/fluent-bit > fluent-bit-values.yaml
```

```
helm -n monitoring install fluent-bit fluent/fluent-bit -f fluent-bit-values.yaml
```


```
helm -n monitoring upgrade fluent-bit fluent/fluent-bit -f fluent-bit-values.yaml
```

Get Fluent Bit build information by running these commands:

```
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=fluent-bit,app.kubernetes.io/instance=fluent-bit" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 2020:2020
curl http://127.0.0.1:2020
```

```
kubectl -n monitoring logs $(kubectl get pod -n monitoring | awk '/fluent-bit-/{print $1}') -f
```
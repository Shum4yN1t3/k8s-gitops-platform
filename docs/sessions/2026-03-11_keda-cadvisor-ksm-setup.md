# KEDA + Monitoring

**Дата:** 11.03.2026  
**Кластер:** k8master + k8worker2, Kubernetes v1.35.0  
**GitOps:** ArgoCD v3.3.0

---

## 1. Цель сессии

Задеплоить KEDA в кластер через ArgoCD и настроить автоскейлинг `project-test` deployment по CPU метрике из Prometheus. Попутно починить мониторинг — добавить cAdvisor и kube-state-metrics.

---

## 2. KEDA — установка

### Проблема 1: ArgoCD не мог создать KEDA Application

При попытке добавить KEDA через UI с указанием git репо (`infrastructure/keda/`) и файла `keda-app.yaml` внутри — ArgoCD падал с ошибкой:

```
multiple application sources defined: Helm,Directory
```

**Причина:** ArgoCD видел папку как Directory source, но внутри лежал yaml с helm конфигурацией — конфликт типов source.

**Решение:** KEDA устанавливается как отдельный ArgoCD Application напрямую из helm registry kedacore, без git репо.

### Проблема 2: `keda-issuer.yaml` — невалидный манифест

При попытке создать Application через UI с helm chart `kedacore/keda` версий 2.18.3 и 2.16.1 ArgoCD падал с ошибкой:

```
Failed to unmarshal "keda-issuer.yaml": failed to unmarshal manifest: 
json: offset 2: invalid character '{' looking for beginning of object key string
```

**Причина:** ArgoCD v3.3.0 не мог распарсить `keda-issuer.yaml` внутри helm chart. Проблема воспроизводилась на нескольких версиях chart.

**Решение:** Установить KEDA напрямую через helm, минуя ArgoCD:

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace --version 2.19.0
```

### Итоговая конфигурация KEDA

```bash
helm list -n keda
# NAME   NAMESPACE  REVISION  STATUS    CHART       APP VERSION
# keda   keda       1         deployed  keda-2.19.0  2.19.0

kubectl get pods -n keda
# keda-operator, keda-metrics-apiserver, keda-admission-webhooks — все Running
```

---

## 3. ScaledObject — конфигурация автоскейлинга

Файл: `app/keda-config/scaledobject.yaml`

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: project-test-scaler
  namespace: app
spec:
  scaleTargetRef:
    name: project-test
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc.cluster.local:9090
      threshold: '0.5'
      query: sum(rate(container_cpu_usage_seconds_total{container="project-test"}[1m]))
```

### Проблема 1: Смешанные метрики

В исходном конфиге был указан `metricName: http_requests_total` но query считал CPU — противоречие. Поле `metricName` deprecated, убрали.

### Проблема 2: `unclosed left parenthesis`

KEDA падал с ошибкой:

```
prometheus query api returned error. status: 400
invalid parameter "query": 1:74: parse error: unclosed left parenthesis
```

**Причина:** В yaml файле в репо query был без закрывающей скобки:

```
# Неправильно
sum(rate(container_cpu_usage_seconds_total{container="project-test"}[1m])

# Правильно  
sum(rate(container_cpu_usage_seconds_total{container="project-test"}[1m]))
```

Три закрывающих скобки `)))` в конце.

### Deployment — убрать replicas

Когда KEDA управляет скейлингом — поле `replicas` в Deployment конфликтует. Убрали:

```yaml
spec:
  # replicas: убрать — KEDA управляет сам
  selector:
    matchLabels:
      app: project-test
```

---

## 4. Prometheus — добавление cAdvisor scrape

### Проблема: метрика `container_cpu_usage_seconds_total` не собиралась

KEDA получал `0` по метрике, HPA показывал `0/500m`. Запрос из пода возвращал пустой result:

```bash
kubectl run curl-test --image=curlimages/curl -it --rm --restart=Never -- \
  curl -g 'http://prometheus.monitoring.svc.cluster.local:9090/api/v1/query?query=container_cpu_usage_seconds_total{container="project-test"}'
# {"status":"success","data":{"resultType":"vector","result":[]}}
```

**Причина:** `container_cpu_usage_seconds_total` — cAdvisor метрика, экспортируется kubelet'ом. В `prometheus-config` не было job'а для scrape kubelet/cadvisor.

**Решение:** Добавить `kubernetes-cadvisor` job в `infrastructure/monitoring/02-config.yml`:

```yaml
- job_name: 'kubernetes-cadvisor'
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: true
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  kubernetes_sd_configs:
    - role: node
  relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
    - target_label: __address__
      replacement: kubernetes.default.svc.cluster.local:443
    - source_labels: [__meta_kubernetes_node_name]
      target_label: __metrics_path__
      replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

После применения и рестарта Prometheus:

```bash
kubectl rollout restart deployment prometheus -n monitoring
```

Метрика появилась, KEDA начал получать данные.

---

## 5. Prometheus Rules — ложные алерты

### Проблема: `InstanceDown` срабатывал на sidecar контейнеры

После добавления `kubernetes-pods` scrape job Prometheus начал скрейпить все поды включая init/sidecar контейнеры (`fix-permissions`, `promtail`) которые не экспортируют `/metrics`. Результат — `up=0` и алерты.

**Решение:** Добавить фильтр по job в правиле. Файл `infrastructure/monitoring/07-prom-rules.yml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  availability.yml: |
    groups:
    - name: availability
      rules:
      - alert: InstanceDown
        expr: up{job="node-exporter"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Сервер {{ $labels.instance }} упал"
          description: "Node Exporter перестал отдавать метрики."
```

---

## 6. kube-state-metrics — добавление

Grafana dashboard 315 показывал "No data" для Pods CPU/Memory/Network.

**Причина:** Нет `kube-state-metrics` — источника метрик `kube_pod_*`, `kube_deployment_*`.

Добавлен файл `infrastructure/monitoring/10-kube-stat-metrics.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
        - name: kube-state-metrics
          image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.13.0
          ports:
            - containerPort: 8080
              name: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    app: kube-state-metrics
spec:
  selector:
    app: kube-state-metrics
  ports:
    - port: 8080
      targetPort: 8080
      name: metrics
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
  - apiGroups: [""]
    resources: ["nodes", "pods", "services", "endpoints", "persistentvolumeclaims", "replicationcontrollers", "limitranges", "namespaces"]
    verbs: ["list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
    verbs: ["list", "watch"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
subjects:
  - kind: ServiceAccount
    name: kube-state-metrics
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: kube-state-metrics
  apiGroup: rbac.authorization.k8s.io
```

Scrape job в `02-config.yml`:

```yaml
- job_name: 'kube-state-metrics'
  kubernetes_sd_configs:
    - role: endpoints
  relabel_configs:
    - source_labels: [__meta_kubernetes_service_label_app]
      action: keep
      regex: kube-state-metrics
```

---

## 7. Node Exporter — toleration для control-plane

DaemonSet node-exporter запускался только на `k8worker2` — на мастере стоит taint `node-role.kubernetes.io/control-plane:NoSchedule`.

Добавлен toleration в `infrastructure/monitoring/04-node-exporter.yml`:

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
```

---

## 8. Итоговое состояние

```bash
kubectl get scaledobject -n app
# NAME                  SCALETARGETKIND      MIN  MAX  READY  ACTIVE
# project-test-scaler   apps/v1.Deployment   1    5    True   Unknown

kubectl get hpa -n app
# NAME                           TARGETS        MINPODS  MAXPODS  REPLICAS
# keda-hpa-project-test-scaler   27m/500m(avg)  1        5        1

kubectl get pods -n keda
# keda-operator                Running
# keda-metrics-apiserver       Running
# keda-admission-webhooks      Running

kubectl get pods -n monitoring
# prometheus      Running
# grafana         Running
# alertmanager    Running
# kube-state-metrics  Running
# node-exporter   Running (на обеих нодах после toleration)
# loki            Running
# promtail        Running
```

### Рабочие Grafana dashboards

| ID    | Название               | Требует            |
|-------|------------------------|--------------------|
| 315   | Kubernetes cluster     | cAdvisor           |
| 1860  | Node Exporter Full     | node-exporter      |
| 13332 | Kubernetes Pods        | kube-state-metrics |
| 15760 | Kubernetes Views Global| kube-state-metrics |

---

## 9. Команды для диагностики

```bash
# Проверить метрику в Prometheus
kubectl run curl-test --image=curlimages/curl -it --rm --restart=Never -- \
  curl -g 'http://prometheus.monitoring.svc.cluster.local:9090/api/v1/query?query=container_cpu_usage_seconds_total{container="project-test"}'

# Смотреть скейлинг в реальном времени
kubectl get hpa -n app -w

# Нагрузочный тест
kubectl run load-test --image=busybox --rm -it --restart=Never -- \
  sh -c "while true; do wget -q -O- http://project-test.app.svc.cluster.local:8080/; done"

# Статус ScaledObject
kubectl describe scaledobject project-test-scaler -n app

# Рестарт Prometheus после изменения конфига
kubectl rollout restart deployment prometheus -n monitoring
```
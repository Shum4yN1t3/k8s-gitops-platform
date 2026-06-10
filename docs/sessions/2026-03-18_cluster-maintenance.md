# Техническое обслуживание кластера — оптимизация и устранение проблем

**Дата:** 18.03.2026  
**Кластер:** k8master (203.0.113.10) + k8worker2 (203.0.113.20)  
**Контекст:** Плановая диагностика и оптимизация после инцидентов 17.03.2026

---

## 1. Проблема — cert-manager-cainjector CrashLoopBackOff

### Обнаружение

При проверке всех подов после рестарта мастера обнаружил:

```
cert-manager   cert-manager-cainjector-585f77dbdd-m6x8m   0/1   CrashLoopBackOff   106 (2m50s ago)
```

106 рестартов — под падал всю ночь.

### Диагностика

```bash
kubectl logs -n cert-manager -l app.kubernetes.io/name=cainjector | tail -20
```

Вывод:
```
E0318 06:52:40.541079  1 main.go:45] "cert-manager: error executing command"
err="failed to get API group resources: unable to retrieve the complete list of server APIs:
apiextensions.k8s.io/v1: Get \"https://10.96.0.1:443/apis/apiextensions.k8s.io/v1\":
dial tcp 10.96.0.1:443: i/o timeout"
```

### Причина

Та же что и у ingress-nginx — Calico проверяет NetworkPolicy **после DNAT**. Пакет идёт на `10.96.0.1:443`, kube-proxy делает DNAT на `203.0.113.10:6443`, Calico видит реальный IP и блокирует — правила для него не было.

### Решение

Добавил в `infrastructure/security/network-policies/cert-manager-netpol.yaml` egress на реальный IP мастера для всех трёх компонентов:

```yaml
# Добавлено в cert-manager-allow, cert-manager-webhook-allow, cert-manager-cainjector-allow
- to:
    - ipBlock:
        cidr: 203.0.113.10/32
  ports:
    - protocol: TCP
      port: 6443
```

После коммита и применения ArgoCD удалил зависший под:

```bash
kubectl delete pod -n cert-manager -l app.kubernetes.io/name=cainjector
```

Вывод после восстановления:
```
I0318 06:56:26.514432  1 start.go:193] "starting" version="v1.13.3"
I0318 06:57:47.112418  1 controller.go:220] "cert-manager: Starting workers" worker count=1
I0318 06:57:47.118726  1 reconciler.go:142] "cert-manager: Updated object"

NAME                                       READY   STATUS    RESTARTS
cert-manager-8bf7cd65c-dc5rg               1/1     Running   0
cert-manager-cainjector-585f77dbdd-x6jvx   1/1     Running   0
cert-manager-webhook-6bc56dcb94-57p6t      1/1     Running   0
```

---

## 2. Проблема — шумный мониторинг: ложные алерты

### Обнаружение

Получил алерты в Telegram:

```
🔥 АЛАРМ: warning
Событие: PrometheusTargetDown
Объект: Cluster
Начало: 06:44:00 18.03 UTC

🔥 АЛАРМ: warning
Событие: ContainerHighMemoryUsage
Объект: Cluster
Начало: 06:48:54 18.03 UTC
```

### Диагностика

**ContainerHighMemoryUsage** срабатывал на все контейнеры включая системные:

```bash
kubectl exec -n monitoring deploy/prometheus -- wget -qO- \
  'http://localhost:9090/api/v1/alerts' 2>/dev/null | python3 -m json.tool | grep "container"
```

Вывод показал алерты на: `etcd`, `kube-apiserver`, `kube-controller-manager`, `calico-node`, `kube-proxy`, `node-exporter` и другие системные контейнеры — у всех нет memory limits.

**Причина:** при `container_spec_memory_limit_bytes = 0` формула `working_set / limit * 100` даёт бесконечность → алерт всегда firing.

**PrometheusTargetDown** срабатывал на поды без `/metrics` эндпоинта — `kubernetes-pods` job скрейпит все поды включая те что не экспортируют метрики.

### Решение

Исправил правила в `infrastructure/monitoring/07-prom-rules.yml`:

**ContainerHighMemoryUsage** — добавил фильтр `> 0`:

```yaml
- alert: ContainerHighMemoryUsage
  expr: |
    (container_memory_working_set_bytes{container!="", container!="POD"}
    / container_spec_memory_limit_bytes{container!="", container!="POD"} * 100 > 90)
    and container_spec_memory_limit_bytes{container!="", container!="POD"} > 0
  for: 10m
```

**PrometheusTargetDown** — ограничил только важные jobs:

```yaml
- alert: PrometheusTargetDown
  expr: up{job=~"node-exporter|kube-state-metrics|kubernetes-cadvisor"} == 0
  for: 5m
```

После коммита ArgoCD применил изменения автоматически — алерты пропали через 5 минут.

---

## 3. Оптимизация — чистка мёртвых ReplicaSet

### Обнаружение

```bash
kubectl get rs -A | awk '$3==0 && $4==0 && $5==0'
```

Обнаружил десятки мёртвых ReplicaSet — особенно в namespace `monitoring`:

```
monitoring   prometheus-5988d444f    0  0  0  7d19h
monitoring   prometheus-66bf46846   0  0  0  7d19h
monitoring   grafana-56f5857c5      0  0  0  41d
monitoring   grafana-5c644cd96f     0  0  0  40d
# ... ещё десятки
```

### Причина

`revisionHistoryLimit: 10` (дефолт) — Kubernetes хранит 10 последних ReplicaSet для возможности rollback. При частых sync через ArgoCD и рестартах это приводило к накоплению мусора.

### Решение

Почистил все мёртвые RS:

```bash
kubectl get rs -A -o json | python3 -c "
import sys,json
d=json.load(sys.stdin)
for rs in d['items']:
    if rs['spec']['replicas'] == 0:
        print(f\"kubectl delete rs {rs['metadata']['name']} -n {rs['metadata']['namespace']}\")
" | bash
```

Уменьшил `revisionHistoryLimit` до `2` в манифестах репо:

- `app/deployment.yaml`
- `infrastructure/monitoring/03-main-stack.yml`
- `infrastructure/monitoring/06-alertmanager.yml`
- `infrastructure/monitoring/08-loki.yml`
- `infrastructure/monitoring/10-kube-stat-metrics.yaml`

```yaml
spec:
  revisionHistoryLimit: 2  # было 10
```

В GitOps всю историю хранит git — хранить 10 RS нет смысла.

---

## 4. Установка metrics-server

### Проблема

```bash
kubectl top nodes
# error: Metrics API not available

kubectl top pods -A
# error: Metrics API not available
```

`kubectl top` не работал — metrics-server не был установлен. Без него также не работает нативный HPA по CPU (хотя KEDA использует Prometheus).

### Установка

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

На bare metal нужен флаг `--kubelet-insecure-tls` так как kubelet не имеет валидного сертификата для metrics-server:

```bash
kubectl patch deployment metrics-server -n kube-system --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

После запуска:

```bash
kubectl top nodes
# NAME        CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
# k8master    257m         4%       3026Mi          25%
# k8worker2   280m         7%       3368Mi          42%
```

---

## 5. Итоговое состояние кластера

```bash
kubectl get pods -A | grep -v Running | grep -v Completed
# Нет проблемных подов

kubectl top nodes
# k8master    257m  4%   3026Mi  25%
# k8worker2   280m  7%   3368Mi  42%

kubectl get applications -n argocd
# NAME          SYNC STATUS   HEALTH STATUS
# app-deploy    Synced        Healthy
# certmanager   Synced        Healthy
# ingressnginx  Synced        Healthy
# monitoring    Synced        Healthy
# security      Synced        Healthy
# vault         Synced        Healthy
# ...все Synced Healthy
```

---

## 6. Выводы и правила на будущее

### NetworkPolicy с Calico — обязательное правило

Любой компонент который обращается к kube-apiserver требует **два egress правила**:

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 10.96.0.1/32    # ClusterIP
    ports:
      - protocol: TCP
        port: 443
  - to:
      - ipBlock:
          cidr: 203.0.113.10/32  # Реальный IP мастера (после DNAT)
    ports:
      - protocol: TCP
        port: 6443
```

Компоненты которым это нужно: cert-manager, ingress-nginx, keda, vault-injector, metallb, kube-state-metrics.

### Правила для Prometheus алертов

- Всегда добавлять `and metric > 0` когда делишь на метрику которая может быть 0
- `PrometheusTargetDown` фильтровать по конкретным jobs, не по всем `up == 0`
- Тестировать новые правила в Prometheus UI перед коммитом

### revisionHistoryLimit

В GitOps кластере ставить `2` вместо дефолтных `10`. Rollback делается через git revert, не через Kubernetes rollout.

### metrics-server на bare metal

Всегда добавлять `--kubelet-insecure-tls` аргумент при установке на bare metal кластер.

---

## 7. Команды для регулярного обслуживания

```bash
# Проверить мёртвые ReplicaSet
kubectl get rs -A | awk 'NR>1 && $3==0 && $4==0 && $5==0 {print $1, $2}'

# Почистить мёртвые RS
kubectl get rs -A -o json | python3 -c "
import sys,json
d=json.load(sys.stdin)
for rs in d['items']:
    if rs['spec']['replicas'] == 0:
        print(f\"kubectl delete rs {rs['metadata']['name']} -n {rs['metadata']['namespace']}\")
" | bash

# Проверить здоровье кластера
kubectl get pods -A | grep -v Running | grep -v Completed | grep -v Succeeded

# Проверить ресурсы
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -20

# Проверить firing алерты
kubectl exec -n monitoring deploy/prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/alerts' 2>/dev/null | \
  python3 -c "
import sys,json
d=json.load(sys.stdin)
alerts=[a for a in d['data']['alerts'] if a['state']=='firing']
print(f'Firing: {len(alerts)}')
[print(f'  {a[\"labels\"][\"alertname\"]} [{a[\"labels\"].get(\"severity\",\"?\")}] - {a[\"labels\"].get(\"container\",a[\"labels\"].get(\"instance\",\"?\"))}') for a in alerts]
"
```

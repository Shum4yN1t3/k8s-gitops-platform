# Monitoring Stack — Production Guide

## Архитектура

```
┌──────────────┐    scrape     ┌──────────────┐    alert     ┌──────────────┐   telegram   ┌──────────┐
│ Node Exporter│───────────────│  Prometheus   │─────────────│ Alertmanager │─────────────│ Telegram │
│ cAdvisor     │               │  :9090        │             │  :9093       │             │   Bot    │
│ KSM          │               └──────┬────────┘             └──────────────┘             └──────────┘
└──────────────┘                      │ datasource
                                      ▼
                               ┌──────────────┐
                               │   Grafana     │
                               │   :3000       │
                               └──────────────┘
```

### Компоненты

| Компонент | Тип | Описание | Источник метрик |
|-----------|-----|----------|-----------------|
| **Prometheus** | Deployment | Сбор и хранение метрик, оценка правил | — |
| **Alertmanager** | Deployment | Маршрутизация и отправка уведомлений | — |
| **Grafana** | Deployment | Визуализация дашбордов | — |
| **Node Exporter** | DaemonSet | Метрики ОС: CPU, RAM, диск, сеть | `node_*` |
| **kube-state-metrics** | Deployment | Метрики K8s объектов: pods, deployments, nodes | `kube_*` |
| **cAdvisor** | Встроен в kubelet | Метрики контейнеров: CPU, RAM, I/O | `container_*` |
| **Loki** | Deployment | Агрегация логов | — |
| **Promtail** | DaemonSet | Сбор логов с узлов и отправка в Loki | — |

### Откуда приходят метрики (scrape jobs)

| Job | Что собирает | Метрики |
|-----|-------------|---------|
| `node-exporter` | OS-метрики каждого узла | `node_cpu_*`, `node_memory_*`, `node_filesystem_*`, `node_disk_*`, `node_network_*` |
| `kube-state-metrics` | Состояние K8s-объектов | `kube_pod_*`, `kube_deployment_*`, `kube_node_*`, `kube_daemonset_*`, `kube_job_*` |
| `kubernetes-cadvisor` | Ресурсы контейнеров | `container_cpu_*`, `container_memory_*`, `container_fs_*` |
| `kubernetes-pods` | Приложения из ns `app` и `monitoring` | Кастомные метрики приложений |

---

## Файлы манифестов

```
infrastructure/monitoring/
├── 01-rbac.yml                # ClusterRole для Prometheus
├── 02-config.yml              # ConfigMap: prometheus.yml (scrape + alerting config)
├── 03-main-stack.yml          # Deployment: Prometheus + Grafana
├── 04-node-exporter.yml       # DaemonSet: Node Exporter
├── 05-ingress-grafana.yml     # Ingress: Grafana + Prometheus
├── 06-alertmanager.yml        # ConfigMap + Deployment: Alertmanager + Templates
├── 07-prom-rules.yml          # ConfigMap: все правила алертинга (production)
├── 08-loki.yml                # Deployment: Loki
├── 09-promtail.yml            # DaemonSet: Promtail
├── 10-kube-stat-metrics.yaml  # Deployment + RBAC: kube-state-metrics
├── grafana-sealed-secret.yaml # SealedSecret: пароль Grafana
├── promtail-rbac.yml          # RBAC для Promtail
├── kustomization.yaml         # Kustomize: список ресурсов
└── rules/                     # Справочные копии правил (не монтируются)
    ├── availability.yml
    ├── app_monitoring.yml
    └── telegram_alert.tmpl
```

---

## Как устроен алертинг

### Цепочка доставки алерта

```
1. Prometheus оценивает expr каждые 15s (scrape_interval)
2. Если expr == true в течение `for` → алерт переходит в FIRING
3. Prometheus отправляет алерт на Alertmanager
4. Alertmanager группирует по alertname+namespace
5. Alertmanager рендерит Go-шаблон telegram_alert.tmpl
6. Alertmanager отправляет сообщение в Telegram через Bot API
7. Когда expr снова false → RESOLVED (send_resolved: true)
```

### Маршрутизация в Alertmanager

| Severity | group_wait | group_interval | repeat_interval | Описание |
|----------|-----------|----------------|-----------------|----------|
| `critical` | 10s | 1m | 1h | Немедленная отправка, повтор каждый час |
| `warning` | 30s | 5m | 4h | Отложенная отправка, повтор каждые 4 часа |

### Что такое `for`

- `for: 0m` — алерт срабатывает мгновенно (например, OOMKilled)
- `for: 2m` — expr должен быть true 2 минуты подряд (фильтрует кратковременные всплески)
- `for: 10m` — для "мягких" предупреждений (CPU >85%)

---

## Группы правил

### 1. `node_health` — Здоровье узлов

| Алерт | Выражение | Severity | Для чего |
|-------|-----------|----------|----------|
| NodeExporterDown | `up{job="node-exporter"} == 0` | critical | Node Exporter не отвечает |
| NodeExporterMissing | `absent(up{job="node-exporter"})` | critical | Нет ни одного NE в кластере |
| NodeHighCpuUsage | CPU idle <15% (10m) | warning | Высокая загрузка CPU |
| NodeCriticalCpuUsage | CPU idle <5% (5m) | critical | Критическая загрузка CPU |
| NodeHighMemoryUsage | RAM >85% (10m) | warning | Мало свободной памяти |
| NodeCriticalMemoryUsage | RAM >95% (5m) | critical | Почти нет памяти |
| NodeDiskAlmostFull | Диск >80% (15m) | warning | Диск заполняется |
| NodeDiskCriticallyFull | Диск >90% (5m) | critical | Диск почти полон |
| NodeDiskWillFillIn24h | `predict_linear` | warning | Предсказание заполнения |
| NodeHighDiskIOUtilization | IO util >90% (15m) | warning | Диск перегружен |
| NodeNetworkErrors | >10 err/s (5m) | warning | Сетевые ошибки |
| NodeClockSkew | offset >50ms (10m) | warning | Рассинхрон часов |
| NodeHighLoad | LA15 >2x CPU (15m) | warning | Высокий Load Average |

### 2. `kubernetes_workloads` — Поды и деплойменты

| Алерт | Severity | Для чего |
|-------|----------|----------|
| PodCrashLooping | critical | >3 рестарта за 15 минут |
| PodNotReady | warning | Pod не Ready 5+ минут |
| ContainerOOMKilled | critical | Контейнер убит по OOM |
| ContainerWaiting | warning | Контейнер завис (ImagePull, CrashLoop) |
| DeploymentReplicasMismatch | warning | Готовых реплик меньше желаемых |
| DeploymentGenerationMismatch | warning | Rollout завис |
| DaemonSetNotScheduled | warning | Поды DS не размещены |
| DaemonSetMisScheduled | warning | Поды DS на неправильных узлах |
| StatefulSetReplicasMismatch | warning | STS не может набрать реплики |
| JobFailed | warning | Job завершился с ошибкой |
| JobStuck | warning | Job работает >1 часа |

### 3. `kubernetes_nodes` — Состояние узлов K8s

| Алерт | Severity | Для чего |
|-------|----------|----------|
| KubeNodeNotReady | critical | Узел не Ready (kubelet не отвечает) |
| KubeNodeDiskPressure | critical | Kubelet видит нехватку диска |
| KubeNodeMemoryPressure | critical | Kubelet видит нехватку памяти |
| KubeNodePIDPressure | warning | Заканчиваются PID на узле |

### 4. `container_resources` — Ресурсы контейнеров

| Алерт | Severity | Для чего |
|-------|----------|----------|
| ContainerHighCpuThrottling | warning | CPU throttling >50% (нужно больше CPU limits) |
| ContainerHighMemoryUsage | warning | >90% от memory limit (близок к OOM) |

### 5. `app_alerts` — Приложение (namespace: app)

| Алерт | Severity | Для чего |
|-------|----------|----------|
| AppInstanceMissing | critical | В ns `app` нет ни одного пода |
| AppInstanceDown | critical | Поды в ns `app` не отвечают |

### 6. `self_monitoring` — Мониторинг мониторинга

| Алерт | Severity | Для чего |
|-------|----------|----------|
| PrometheusTargetDown | warning | Любой scrape target недоступен |
| PrometheusConfigReloadFailed | critical | Prometheus не загрузил конфиг |
| AlertmanagerConfigReloadFailed | critical | Alertmanager не загрузил конфиг |
| PrometheusRuleEvaluationFailures | warning | Ошибки в PromQL-выражениях |
| AlertmanagerNotificationsFailing | critical | Telegram-бот не может отправить |
| KubeStateMetricsDown | critical | KSM недоступен (без него нет kube_* метрик) |

---

## Как добавить новое правило

### Шаг 1 — Выбрать источник метрик

Определите, откуда берётся метрика:

```
node_*             → Node Exporter (метрики ОС)
kube_*             → kube-state-metrics (объекты K8s)
container_*        → cAdvisor через kubelet (ресурсы контейнеров)
up{job="..."}      → Prometheus (доступность scrape target)
prometheus_*       → Prometheus (self-метрики)
alertmanager_*     → Alertmanager (self-метрики)
```

### Шаг 2 — Проверить выражение в Prometheus UI

Откройте `https://prometheus.example.com` и проверьте PromQL:

```promql
# Пример: проверить свободную память на узлах
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

### Шаг 3 — Добавить правило в ConfigMap

Отредактируйте `07-prom-rules.yml`. Найдите нужную группу и добавьте правило:

```yaml
      - alert: MyNewAlert
        expr: my_metric > threshold
        for: 5m             # сколько держать перед FIRING
        labels:
          severity: warning  # или critical
        annotations:
          summary: "Краткое описание для {{ $labels.instance }}"
          description: "Подробное описание. Значение: {{ $value | printf \"%.1f\" }}"
```

### Шаг 4 — Запушить в git

ArgoCD автоматически синхронизирует ConfigMap. Prometheus перечитает правила.

```bash
git add infrastructure/monitoring/07-prom-rules.yml
git commit -m "alert: add MyNewAlert rule"
git push
```

### Шаг 5 — Проверить загрузку

```bash
# Проверить что правило загружено
kubectl exec -n monitoring deploy/prometheus -- \
  wget -qO- http://localhost:9090/api/v1/rules | python3 -m json.tool | grep MyNewAlert
```

---

## Анатомия правила (best practices)

```yaml
- alert: NodeHighCpuUsage              # PascalCase, описательное имя
  expr: |                               # PromQL-выражение
    100 - (avg by(instance)
      (rate(node_cpu_seconds_total{mode="idle"}[5m]))
    * 100) > 85
  for: 10m                             # "мягкий" порог — ждём 10 минут
  labels:
    severity: warning                   # warning или critical
    # НЕ добавляйте сюда instance/pod — они приходят из метрики
  annotations:
    summary: "Краткое описание"         # одна строка для заголовка
    description: "Подробности"          # для тела сообщения
```

### Best Practices для правил

1. **Два уровня severity**: `warning` (внимание) и `critical` (срочное действие)
2. **`for` время**: critical — 2-5m, warning — 5-15m. Не ставьте `for: 0m` без причины
3. **Используйте `rate()` для counter-метрик**: `rate(metric[5m])`, не `metric`
4. **Фильтруйте шум**: исключайте `tmpfs`, `lo`, `veth*` из disk/network алертов
5. **`absent()` для критических компонентов**: если метрика исчезает — это тоже проблема
6. **`predict_linear()` для disk**: предсказать заполнение заранее
7. **Не алертите на то, что не можете исправить**: каждый алерт = действие
8. **Описания на русском**: ваша команда должна понимать алерт за 3 секунды

### Что НЕ стоит алертить

- Кратковременные всплески CPU/RAM (используйте `for`)
- Метрики без actionable действия
- Слишком низкие пороги (>50% CPU — это нормально)
- Дубликаты одного и того же инцидента

---

## Как работает шаблон Telegram

Файл `telegram_alert.tmpl` — это Go template. Alertmanager рендерит его для каждой группы алертов.

### Доступные переменные

```
{{ .Status }}                    → "firing" или "resolved"
{{ .CommonLabels.alertname }}    → имя алерта
{{ .CommonLabels.severity }}     → severity из labels
{{ .CommonLabels.namespace }}    → namespace из метрики
{{ .CommonAnnotations.summary }} → summary из annotations
{{ .CommonAnnotations.description }}
{{ .Alerts.Firing | len }}       → количество активных алертов
{{ .Alerts.Resolved | len }}     → количество решённых
{{ (index .Alerts 0).StartsAt }} → время начала (Go time.Time)
```

### Формат сообщения

```
🔥 АЛАРМ [CRITICAL]

Алерт: NodeCriticalCpuUsage
Namespace: infrastructure
Объект: 10.0.0.5:9100

Описание: CPU загружен на 97.3% более 5 минут.

Статус: 🟥 FIRING
Активных: 1 | Решено: 0
Начало: 14:23:05 17.03.2026 UTC
```

---

## Проверка работоспособности (Runbook)

### 1. ArgoCD синхронизация

```bash
# Проверить статус приложения monitoring
kubectl get app monitoring -n argocd -o jsonpath='{.status.sync.status}'
# Ожидается: Synced

# Проверить health
kubectl get app monitoring -n argocd -o jsonpath='{.status.health.status}'
# Ожидается: Healthy

# Принудительный sync если нужно
kubectl patch app monitoring -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
```

### 2. Проверить что все поды Running

```bash
kubectl get pods -n monitoring
# Все должны быть Running/Ready
```

### 3. Проверить что Prometheus загрузил правила

```bash
# Количество загруженных групп правил (ожидается 6)
kubectl exec -n monitoring deploy/prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/rules' 2>/dev/null | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Groups: {len(d[\"data\"][\"groups\"])}'); [print(f'  {g[\"name\"]}: {len(g[\"rules\"])} rules') for g in d['data']['groups']]"
```

### 4. Проверить подключение к Alertmanager

```bash
# Alertmanager должен быть в targets
kubectl exec -n monitoring deploy/prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/alertmanagers' 2>/dev/null | python3 -m json.tool
```

### 5. Проверить что Alertmanager загрузил шаблон

```bash
# Статус конфига
kubectl exec -n monitoring deploy/alertmanager -- \
  wget -qO- 'http://localhost:9093/api/v2/status' 2>/dev/null | python3 -m json.tool
```

### 6. Тестовая отправка в Telegram (вручную)

```bash
# Прямой тест бота — отправит сообщение в ваш чат
curl -s -X POST "https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/sendMessage" \
  -d chat_id=<TELEGRAM_CHAT_ID> \
  -d parse_mode=HTML \
  -d text="✅ <b>Тест:</b> Alertmanager Telegram интеграция работает"
```

### 7. Тест через Alertmanager API (имитация алерта)

```bash
# Отправить фейковый алерт напрямую в Alertmanager
kubectl exec -n monitoring deploy/alertmanager -- \
  wget -qO- --post-data='[{
    "labels": {
      "alertname": "TestAlert",
      "severity": "critical",
      "instance": "test-instance"
    },
    "annotations": {
      "summary": "Тестовый алерт",
      "description": "Это тестовое сообщение для проверки доставки в Telegram."
    },
    "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
    "generatorURL": "http://prometheus:9090"
  }]' \
  --header='Content-Type: application/json' \
  'http://localhost:9093/api/v2/alerts'
```

### 8. Проверить активные алерты

```bash
# Текущие firing алерты в Prometheus
kubectl exec -n monitoring deploy/prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/alerts' 2>/dev/null | \
  python3 -c "import sys,json; d=json.load(sys.stdin); alerts=[a for a in d['data']['alerts'] if a['state']=='firing']; print(f'Firing: {len(alerts)}'); [print(f'  {a[\"labels\"][\"alertname\"]} [{a[\"labels\"].get(\"severity\",\"?\")}]') for a in alerts]"

# Текущие алерты в Alertmanager
kubectl exec -n monitoring deploy/alertmanager -- \
  wget -qO- 'http://localhost:9093/api/v2/alerts' 2>/dev/null | python3 -m json.tool
```

---

## Troubleshooting

### Алерты не приходят в Telegram

```bash
# 1. Проверить что alertmanager запущен
kubectl get pods -n monitoring -l app=alertmanager

# 2. Проверить логи alertmanager
kubectl logs -n monitoring deploy/alertmanager --tail=50

# 3. Проверить что шаблон примонтирован
kubectl exec -n monitoring deploy/alertmanager -- ls /etc/alertmanager/templates/
# Ожидается: telegram_alert.tmpl

# 4. Проверить что Telegram бот доступен из кластера
kubectl exec -n monitoring deploy/alertmanager -- \
  wget -qO- 'https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/getMe'

# 5. Проверить failed notifications
kubectl exec -n monitoring deploy/alertmanager -- \
  wget -qO- 'http://localhost:9093/metrics' 2>/dev/null | grep alertmanager_notifications_failed
```

### Prometheus не загрузил правила

```bash
# 1. Проверить логи Prometheus
kubectl logs -n monitoring deploy/prometheus --tail=50 | grep -i "rule\|error\|warn"

# 2. Проверить что файлы правил на месте
kubectl exec -n monitoring deploy/prometheus -- ls /etc/prometheus/rules/
# Ожидается: node_health.yml, kubernetes_workloads.yml, ...

# 3. Проверить конфиг reload
kubectl exec -n monitoring deploy/prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/status/config' 2>/dev/null | python3 -m json.tool | head -20

# 4. Принудительный reload конфига
kubectl exec -n monitoring deploy/prometheus -- \
  wget -qO- --post-data='' 'http://localhost:9090/-/reload'
```

### kube-state-metrics не отдаёт метрики

```bash
# 1. Проверить под
kubectl get pods -n monitoring -l app=kube-state-metrics

# 2. Проверить что метрики доступны
kubectl exec -n monitoring deploy/kube-state-metrics -- \
  wget -qO- 'http://localhost:8080/metrics' 2>/dev/null | head -20

# 3. Проверить RBAC
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:kube-state-metrics
```

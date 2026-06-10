# Vaultwarden, внешний мониторинг, уведомления

**Дата:** 09.04.2026  
**Кластер:** k8master + k8worker2  
**Контекст:** Деплой Vaultwarden, расширение мониторинга внешних сервисов, настройка уведомлений через ArgoCD Notifications / Alertmanager / GitHub Actions.

---

## 1. Реструктуризация репозитория

### Проблема

Все манифесты приложений лежали в корне `apps/` как отдельные YAML-файлы ArgoCD Application. Не было места для вспомогательных манифестов конкретного приложения (Ingress, StatefulSet, Service и т.д.).

### Решение

Манифесты приложений переехали в подпапки `apps/<appname>/`. ArgoCD Application указывает на эту папку.

**Структура до:**
```
apps/
  vaultwarden.yaml     ← ArgoCD Application
```

**Структура после:**
```
apps/
  vaultwarden.yaml     ← ArgoCD Application (path: apps/warden)
  warden/
    kustomization.yaml
    statefulset.yaml
    service.yaml
    ingress.yaml
    networkpolicy.yaml
    limitrange.yaml
    resourcequota.yaml
```

### Исправление CI после реструктуризации

`kustomization.yaml` внутри `apps/warden/` валидировался kubeconform как k8s ресурс и падал. Фикс в `.github/workflows/lint.yml`:

```yaml
- name: Validate ArgoCD application definitions
  run: |
    kubeconform -summary \
      -schema-location default \
      -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
      -skip Application \
      -ignore-filename-pattern '.*kustomization.*' \   # ← добавлено
      apps/
```

---

## 2. Vaultwarden

**URL:** `https://warden.example.com`  
**Image:** `vaultwarden/server:1.32.7`  
**Namespace:** `vaultwarden`

### Файлы

| Файл | Описание |
|------|----------|
| `apps/vaultwarden.yaml` | ArgoCD Application |
| `apps/warden/statefulset.yaml` | StatefulSet + PVC 5Gi |
| `apps/warden/ingress.yaml` | Ingress с cert-manager TLS |
| `apps/warden/networkpolicy.yaml` | default-deny + allow ingress-nginx |
| `apps/warden/limitrange.yaml` | LimitRange namespace |
| `apps/warden/resourcequota.yaml` | ResourceQuota namespace |

### StatefulSet — ключевые параметры

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        runAsGroup: 65534
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: vaultwarden
          image: vaultwarden/server:1.32.7
          ports:
            - containerPort: 8080
          env:
            - name: ROCKET_PORT
              value: "8080"
            - name: SIGNUPS_ALLOWED
              value: "true"
            - name: DOMAIN
              value: https://warden.example.com
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: local-path
        resources:
          requests:
            storage: 5Gi
```

### Ingress

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  rules:
    - host: warden.example.com
```

### PSA namespace label

В `infrastructure/security/psa/namespace-labels.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vaultwarden
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

### Network Policy — критичный фикс

Без этого правила ingress-nginx не мог достучаться до vaultwarden.  
Файл: `infrastructure/security/network-policies/ingress-nginx-netpol.yaml`

```yaml
# Добавить в egress секцию ingress-nginx политики:
- to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: vaultwarden
  ports:
    - protocol: TCP
      port: 8080
```

---

## 3. Мониторинг внешних сервисов

### Blackbox Exporter

**Image:** `prom/blackbox-exporter:v0.25.0`  
**Port:** 9115  
**Файл:** `infrastructure/monitoring/11-blackbox-exporter.yml`

```yaml
resources:
  requests:
    cpu: 25m      # минимум по LimitRange
    memory: 32Mi
```

Добавлены scrape jobs в `infrastructure/monitoring/02-config.yml`:

| Job | Цели |
|-----|------|
| `blackbox-own-services` | warden, argocd, vault, www.example.org, git.example.org, mail.example.org |
| `blackbox-infrastructure` | Cloudflare, Let's Encrypt, Google DNS |
| `blackbox-dev-tools` | GitHub, GitLab, Docker Hub, npm, PyPI, Helm |
| `blackbox-messaging` | Telegram, Discord, Slack |
| `blackbox-cloud` | AWS, Azure, GCP, Vercel, HashiCorp |

### JSON Exporter (статусные страницы)

**Image:** `prometheuscommunity/json-exporter:v0.6.0`  
**Port:** 7979  
**Файл:** `infrastructure/monitoring/12-json-exporter.yml`

Парсит Statuspage API (`/api/v2/status.json`):

```yaml
# ConfigMap конфигурация:
modules:
  statuspage:
    metrics:
      - name: status_indicator
        type: gauge
        path: "{.status.indicator}"
        string_value_mapping:
          map:
            none: 0
            minor: 1
            major: 2
            critical: 3
```

Цели: Cloudflare, GitHub, Discord, Vercel.

Readiness probe — tcpSocket (не httpGet, т.к. `/healthz` отсутствует):

```yaml
readinessProbe:
  tcpSocket:
    port: 7979
```

### Network Policy для мониторинга

Добавлены правила в `infrastructure/security/network-policies/monitoring-netpol.yaml`:

```yaml
# blackbox-exporter: исходящий интернет 80/443
- podSelector:
    matchLabels:
      app: blackbox-exporter
  egress:
    - to: []           # internet
      ports:
        - port: 80
        - port: 443
    - to: [DNS]        # kube-dns

# json-exporter: исходящий интернет 443
- podSelector:
    matchLabels:
      app: json-exporter
  egress:
    - to: []
      ports:
        - port: 443
    - to: [DNS]
```

### PSA — фикс для Promtail

Promtail требует `hostPath` и privileged контейнер. При `enforce: baseline` поды не планировались.

В `infrastructure/security/psa/namespace-labels.yaml`:

```yaml
# До:
pod-security.kubernetes.io/enforce: baseline

# После:
pod-security.kubernetes.io/enforce: privileged
```

### Alert Rules

Файл: `infrastructure/monitoring/rules/availability.yml` (и встроен в `07-prom-rules.yml`).

| Alert | Условие | Severity |
|-------|---------|----------|
| `ExternalServiceDown` | `probe_success == 0` 5 мин | critical |
| `OwnServiceDown` | собственные домены недоступны 3 мин | critical |
| `SSLCertExpiringIn14Days` | SSL < 14 дней | warning |
| `SSLCertExpiringIn3Days` | SSL < 3 дней | critical |
| `ExternalServiceStatusDegraded` | statuspage minor/major | warning |
| `ExternalServiceStatusCritical` | statuspage critical | critical |

---

## 4. ArgoCD Notifications — попытка и отказ

### Что пробовали

Попытка настроить ArgoCD Notifications для Telegram-уведомлений о sync/health ArgoCD apps.

Файлы (удалены в конце сессии):
- `apps/argocd-notifications.yaml` — ArgoCD Application
- `infrastructure/argocd-notifications/configmap.yaml` — configmap с триггерами/шаблонами
- `infrastructure/argocd-notifications/secret.yaml` — SealedSecret с токеном бота

### Проблема

Контроллер ArgoCD v3.3.0 стабильно возвращал `Bad Request: chat not found`, несмотря на:
- Корректный токен бота в секрете
- Верный chat_id
- Рабочую сеть из argocd namespace (проверено debug pod + curl)
- Отсутствие блокирующих NetworkPolicy

```
{"level":"error","msg":"Failed to notify recipient {telegram <TELEGRAM_CHAT_ID>}:
Bad Request: chat not found","resource":"argocd/vaultwarden"}
```

Диагностика:

```bash
# Проверить NetworkPolicy
kubectl get networkpolicy argocd-notifications-controller-network-policy -n argocd -o yaml
# → policyTypes: [Ingress] only — egress не ограничен

# Проверить секрет
kubectl get secret argocd-notifications-secret -n argocd \
  -o jsonpath='{.data.telegram-token}' | base64 -d

# Тест сети из argocd namespace
kubectl run debug-curl --image=curlimages/curl --rm -it --restart=Never -n argocd \
  -- curl -s -X POST "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d "chat_id=<CHAT_ID>" -d "text=test"
# → сработало, сообщение пришло

# Найти бинарник для CLI теста
kubectl exec -n argocd $POD -- sh -c "ls /usr/local/bin/"
# → argocd-notifications есть, но в v3.3.0 это только controller, без CLI команд
```

### Решение

Отказались от ArgoCD Notifications. Вместо него — Alertmanager + GitHub Actions.

Удаление без каскадного удаления ресурсов:

```bash
# Убрать finalizer чтобы удаление Application не удалило ConfigMap/Secret
kubectl patch app argocd-notifications -n argocd \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

# Удалены файлы из репо:
# apps/argocd-notifications.yaml
# infrastructure/argocd-notifications/
```

---

## 5. Уведомления: Alertmanager + GitHub Actions

### 5.1. GitHub Actions → Telegram при push

Добавлен job `notify` в `.github/workflows/lint.yml`:

```yaml
notify:
  name: Notify Telegram
  runs-on: ubuntu-latest
  needs: [yamllint, kubeconform, kustomize]
  if: github.event_name == 'push'
  steps:
    - name: Notify on push
      uses: appleboy/telegram-action@v1.0.0
      with:
        to: ${{ secrets.TELEGRAM_CHAT_ID }}
        token: ${{ secrets.TELEGRAM_BOT_TOKEN }}
        format: html
        message: |
          <b>infra / ${{ github.ref_name }}</b>

          <b>Commit:</b> <code>${{ github.event.head_commit.message }}</code>
          <b>Author:</b> ${{ github.event.head_commit.author.name }}
          <b>SHA:</b> <code>${{ github.sha }}</code>
          <b>CI:</b> all checks passed
```

Требуемые GitHub Secrets:
- `TELEGRAM_BOT_TOKEN` — токен бота
- `TELEGRAM_CHAT_ID` — ID чата

Добавляются в: GitHub → Settings → Secrets and variables → Actions.

### 5.2. Prometheus scrape ArgoCD метрик

Добавлен scrape job в `infrastructure/monitoring/02-config.yml`:

```yaml
- job_name: 'argocd'
  static_configs:
    - targets:
      - argocd-metrics.argocd.svc.cluster.local:8082
      - argocd-server-metrics.argocd.svc.cluster.local:8083
```

### 5.3. Alert Rules для ArgoCD

Добавлены в `infrastructure/monitoring/07-prom-rules.yml` (ConfigMap `prometheus-rules`):

```yaml
argocd.yml: |
  groups:
  - name: argocd
    rules:
    - alert: ArgoCDAppHealthDegraded
      expr: argocd_app_info{health_status="Degraded"} == 1
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "ArgoCD app {{ $labels.name }} health degraded"
        description: "App {{ $labels.name }} in {{ $labels.dest_namespace }} Degraded > 2m"

    - alert: ArgoCDAppSyncError
      expr: increase(argocd_app_sync_total{phase="Error"}[10m]) > 0
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "ArgoCD app {{ $labels.name }} sync error"
        description: "App {{ $labels.name }} had a sync error in the last 10 minutes."

    - alert: ArgoCDAppOutOfSync
      expr: argocd_app_info{sync_status="OutOfSync"} == 1
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "ArgoCD app {{ $labels.name }} out of sync"
        description: "App {{ $labels.name }} OutOfSync > 15 minutes."
```

**Важно:** правила должны быть в ConfigMap `prometheus-rules` (файл `07-prom-rules.yml`), а не в отдельных файлах папки `rules/` — они не монтируются в Prometheus.

### Итоговый flow уведомлений

```
git push (k8s-gitops-platform/main)
  → GitHub Actions CI (lint + validate)
  → Telegram: "<b>infra / main</b> commit: ..."

ArgoCD sync → app health Degraded
  → Prometheus: argocd_app_info{health_status="Degraded"}
  → Alertmanager → Telegram: "🔥 АЛАРМ [CRITICAL] ArgoCDAppHealthDegraded"

External service down
  → Blackbox probe_success == 0
  → Alertmanager → Telegram: "🔥 АЛАРМ [CRITICAL] ExternalServiceDown"
```

---

## 6. Полезные команды диагностики

```bash
# Проверить аннотации уведомлений на app
kubectl get app <name> -n argocd -o jsonpath='{.metadata.annotations}' | python3 -m json.tool

# Сбросить "already sent" кэш на всех приложениях
for app in $(kubectl get apps -n argocd --no-headers -o custom-columns=NAME:.metadata.name); do
  kubectl annotate app $app -n argocd notified.notifications.argoproj.io-
done

# Тест Telegram API из кластера
kubectl run debug-curl --image=curlimages/curl --rm -it --restart=Never -n <namespace> \
  -- curl -s -X POST "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d "chat_id=<ID>" -d "text=test"

# Проверить NetworkPolicy egress
kubectl get networkpolicy -n <namespace>
kubectl get networkpolicy <name> -n <namespace> -o yaml | grep -A10 "policyTypes"

# Найти sealed-secrets controller
kubectl get pods -A | grep sealed

# Запечатать секрет
kubectl create secret generic <name> -n <namespace> \
  --from-literal=<key>=<value> \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace kube-system \
           --controller-name sealed-secrets-controller \
           --format yaml > secret.yaml
```

---

## 7. Итоговые изменения файлов

### Добавлено

| Файл | Описание |
|------|----------|
| `apps/vaultwarden.yaml` | ArgoCD Application для Vaultwarden |
| `apps/warden/kustomization.yaml` | Kustomize для vaultwarden |
| `apps/warden/statefulset.yaml` | StatefulSet + PVC |
| `apps/warden/ingress.yaml` | Ingress с TLS |
| `apps/warden/networkpolicy.yaml` | NetworkPolicy vaultwarden namespace |
| `apps/warden/limitrange.yaml` | LimitRange |
| `apps/warden/resourcequota.yaml` | ResourceQuota |
| `infrastructure/monitoring/11-blackbox-exporter.yml` | Blackbox Exporter deployment |
| `infrastructure/monitoring/12-json-exporter.yml` | JSON Exporter deployment |
| `infrastructure/monitoring/rules/availability.yml` | Alert rules для внешних сервисов |

### Изменено

| Файл | Что изменилось |
|------|----------------|
| `infrastructure/monitoring/02-config.yml` | +5 blackbox scrape jobs, +json-status-pages job, +argocd scrape |
| `infrastructure/monitoring/07-prom-rules.yml` | +argocd.yml rules секция |
| `infrastructure/security/psa/namespace-labels.yaml` | +vaultwarden namespace, monitoring enforce → privileged |
| `infrastructure/security/network-policies/ingress-nginx-netpol.yaml` | +egress к vaultwarden:8080 |
| `infrastructure/security/network-policies/monitoring-netpol.yaml` | +blackbox egress internet, +json-exporter egress internet |
| `.github/workflows/lint.yml` | +notify job с Telegram, +ignore kustomization pattern |

### Удалено

| Файл | Причина |
|------|---------|
| `apps/argocd-notifications.yaml` | Заменён Alertmanager + GitHub Actions |
| `infrastructure/argocd-notifications/` | Полностью удалена директория |
| `infrastructure/monitoring/rules/argocd.yml` | Не монтировался в Prometheus, правила перенесены в ConfigMap |

---

## 8. Выводы и правила на будущее

### ArgoCD Notifications v3.3.0

В версии v3.3.0 убраны CLI-команды (`template notify`, `trigger run`) — бинарник работает только как controller. Диагностика через `argocd-notifications` CLI невозможна напрямую.

### Prometheus Rules

Rules файлы в `infrastructure/monitoring/rules/*.yml` **не монтируются** в Prometheus автоматически. Все правила должны быть в ConfigMap `prometheus-rules` (`07-prom-rules.yml`) в секции `data`.

### SealedSecrets

При удалении SealedSecret — удаляется и plain Secret (ownerReference). Обратная замена: удалить оба, создать новый.

```bash
kubectl delete sealedsecret <name> -n <namespace>
kubectl delete secret <name> -n <namespace>  # если не удалился автоматически
kubectl create secret generic <name> -n <namespace> --from-literal=<key>=<value>
```

### Network Policies для egress

При добавлении компонента, которому нужен интернет (exporter, notifications controller), — явно добавить egress правило. Без него трафик блокируется даже если нет явного `default-deny`.

### PSA и Promtail

Promtail требует `privileged` PSA т.к. использует `hostPath` для логов и privileged security context. Namespace `monitoring` должен иметь `enforce: privileged`.

# Устранение x509 webhook-каскада и сброс сертификатов

**Дата:** 12.04.2026  
**Кластер:** k8master + k8worker2  
**Контекст:** После rollout-restart MetalLB controller (предыдущая сессия) кластер перешёл в деградированное состояние: два ArgoCD Application (`loadbalancer`, `monitoring`) застряли в `OutOfSync/Degraded` из-за x509-ошибок в двух независимых webhook. Параллельно cert-manager не мог обновить TLS-сертификат для `warden.example.com`.

---

## 1. Симптомы

### ArgoCD sync failures

```
# loadbalancer app:
failed calling webhook "l2advertisementvalidationwebhook.metallb.io": Post
"https://webhook-service.metallb-system.svc:443/...": tls: failed to verify
certificate: x509: certificate signed by unknown authority

failed calling webhook "ipaddresspoolvalidationwebhook.metallb.io": ...
x509: certificate signed by unknown authority

# monitoring app:
failed calling webhook "validate.nginx.ingress.kubernetes.io": Post
"https://ingress-nginx-controller-admission.ingress-nginx.svc:443/...":
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

### MetalLB controller CrashLoopBackOff

```
NAME                          READY   STATUS             RESTARTS
controller-5cbf55dd54-rcwcc   1/1     Running            2 (29d ago)
controller-779d9b4df5-v4sf2   0/1     CrashLoopBackOff   361 (3m ago)
```

Pod `controller-779d9b4df5-v4sf2` (созданный нашим `kubectl rollout restart`) не мог пройти readiness-пробу:

```
Readiness probe failed: Get "http://172.16.0.62:7472/metrics": dial tcp
172.16.0.62:7472: connect: connection refused
```

### cert-rotator i/o timeout

Логи из `controller-5cbf55dd54-rcwcc` (старый, Running):

```json
{"level":"error","logger":"cert-rotation","msg":"Error updating webhook with certificate",
"name":"bgppeers.metallb.io","error":"Put \"https://10.96.0.1:443/...\": dial tcp
10.96.0.1:443: i/o timeout"}
```

Cert-rotator пытается обновить caBundle в CRD/VWC → timeout → caBundle остаётся устаревшим → все webhook-обращения падают с x509.

### cert-manager HTTP01 solver rejected

```
Error presenting challenge: pods "cm-acme-http-solver-sgsrv" is forbidden:
minimum cpu usage per Container is 25m, but request is 10m
```

---

## 2. Корневые причины

### Причина 1: NetworkPolicy metallb-system блокировала kube-apiserver

**Файл:** `infrastructure/security/network-policies/metallb-netpol.yaml`

Правило egress для cert-rotator использовало `cidr: 10.96.0.1/32` — это ClusterIP сервиса `kubernetes.default.svc`. Но kube-apiserver запускается с `hostNetwork: true`, поэтому kube-proxy делает DNAT: пакет уходит на реальный IP мастер-ноды (`203.0.113.10:6443`). Calico/iptables проверяют NetworkPolicy **после** DNAT — правило `10.96.0.1/32` никогда не совпадает, трафик отклоняется.

Та же ошибка уже была исправлена в `monitoring-netpol.yaml` (там даже есть комментарий):
```yaml
# API server работает с hostNetwork: true, поэтому ни namespaceSelector,
# ни ipBlock с ClusterIP не матчатся — нужен ipBlock 0.0.0.0/0.
```

**Следствие:** cert-rotator metallb не мог обновить caBundle → mismatch между cert и caBundle в VWC → x509.

### Причина 2: ignoreDifferences покрывало только 2 из 7 webhook-ов MetalLB

**Файл:** `apps/load-balancer.yaml`

Старая конфигурация:
```yaml
ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: ValidatingWebhookConfiguration
    name: metallb-webhook-configuration
    jsonPointers:
      - /webhooks/0/clientConfig/caBundle   # bgppeer
      - /webhooks/1/clientConfig/caBundle   # addresspool
```

У MetalLB **7 webhook-ов** (bgppeer, addresspool, bfdprofile, bgpadvertisement, community, **ipaddresspool**, **l2advertisement**). Когда cert-rotator обновлял caBundle для webhook[5] (ipaddresspool) или webhook[6] (l2advertisement), ArgoCD видел drift и **сносил его обратно в пустое значение** → webhook ломался.

### Причина 3: ingress-nginx-admission VWC без ignoreDifferences

**Файл:** `apps/ingress-nginx.yaml` — не имел `ignoreDifferences` вообще.

**Файл:** `infrastructure/ingress-nginx/install-nginx.yaml` — VWC хранится **без** `caBundle` (поле отсутствует):
```yaml
webhooks:
- clientConfig:
    service:
      name: ingress-nginx-controller-admission
      namespace: ingress-nginx
  # caBundle отсутствует — init-job admission-create патчит его при старте
  failurePolicy: Fail
```

Init-job `admission-create` и `admission-patch` заполняют caBundle во время деплоя. Но ArgoCD с `selfHeal: true` видел поле как "лишнее" и при каждом sync возвращал VWC к версии из git (без caBundle). Webhook становился невалидным → любое изменение Ingress падало с x509.

### Причина 4: LimitRange min.cpu блокировал cert-manager solver

**Файл:** `infrastructure/security/resource-limits/limit-ranges.yaml`

cert-manager создаёт временные ACME HTTP01 solver-поды **в namespace-е Certificate/Ingress** с фиксированным запросом `cpu: 10m`. LimitRange в `monitoring` и `vaultwarden` был настроен с `min.cpu: 25m`. Любая попытка создать solver-под отклонялась с ошибкой forbidden.

Пользователь менял LimitRange вручную (`kubectl edit`), но ArgoCD с `selfHeal: true` немедленно возвращал значение из git (25m). Изменение необходимо было вносить в git-репозиторий.

---

## 3. Исправления в git (коммит `b937a4d`)

### 3.1 metallb-netpol.yaml — разрешить egress к API-серверу

Для controller и speaker — заменить `10.96.0.1/32:443` на `0.0.0.0/0` с портами 443 и 6443:

```yaml
# Было:
- to:
    - ipBlock:
        cidr: 10.96.0.1/32
  ports:
    - protocol: TCP
      port: 443

# Стало (для обоих: metallb-controller-allow и metallb-speaker-allow):
- to:
    - ipBlock:
        cidr: 0.0.0.0/0
  ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 6443
```

### 3.2 apps/load-balancer.yaml — wildcard для всех 7 webhook-ов

Заменить `jsonPointers` на `jqPathExpressions` с wildcard:

```yaml
# Было:
ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: ValidatingWebhookConfiguration
    name: metallb-webhook-configuration
    jsonPointers:
      - /webhooks/0/clientConfig/caBundle
      - /webhooks/1/clientConfig/caBundle

# Стало:
ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: ValidatingWebhookConfiguration
    name: metallb-webhook-configuration
    jqPathExpressions:
      - .webhooks[].clientConfig.caBundle   # все 7 webhook-ов
  - group: apiextensions.k8s.io
    kind: CustomResourceDefinition
    jsonPointers:
      - /status
      - /spec/conversion
      - /spec/conversion/webhook/clientConfig/caBundle
```

### 3.3 apps/ingress-nginx.yaml — добавить ignoreDifferences и retry

```yaml
ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: ValidatingWebhookConfiguration
    name: ingress-nginx-admission
    jqPathExpressions:
      - .webhooks[].clientConfig.caBundle
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
  retry:
    limit: 5
    backoff:
      duration: 30s
      factor: 2
      maxDuration: 5m
```

### 3.4 limit-ranges.yaml — снизить min.cpu до 10m (коммит после b937a4d)

В `monitoring` и `vaultwarden` namespace:

```yaml
# Было:
min:
  cpu: 25m
  memory: 32Mi

# Стало:
min:
  cpu: 10m   # cert-manager ACME HTTP01 solver запрашивает cpu: 10m
  memory: 32Mi
```

---

## 4. Ручные команды для восстановления кластера

После того как ArgoCD применил изменения из git:

### Шаг 1: Проверить, что netpol обновился
```bash
kubectl get networkpolicy -n metallb-system metallb-controller-allow \
  -o jsonpath='{.spec.egress[1].to[0].ipBlock.cidr}{"\n"}'
# Ожидаем: 0.0.0.0/0
```

### Шаг 2: Убить сломанный metallb controller pod
```bash
# Удалить CrashLoopBackOff pod — ReplicaSet создаст новый
kubectl delete pod -n metallb-system controller-779d9b4df5-v4sf2

# Подождать, пока новый pod запустится
kubectl get pods -n metallb-system -w
```

### Шаг 3: Убедиться, что cert-rotator обновил caBundle
```bash
kubectl logs -n metallb-system -l component=controller --tail=30 \
  | grep -E "cert-rotation|Ensuring|error|Cert"
# Ищем: "no cert refresh needed" без последующих "i/o timeout"
```

### Шаг 4: Восстановить ingress-nginx admission cert

Если caBundle в VWC уже пустой (ArgoCD успел сбросить его до нашего фикса):
```bash
# Удалить secret и jobs — ArgoCD при ресинке создаст их заново
kubectl delete secret ingress-nginx-admission -n ingress-nginx
kubectl delete job -n ingress-nginx \
  ingress-nginx-admission-create \
  ingress-nginx-admission-patch --ignore-not-found

# Принудительный ресинк
argocd app sync ingressnginx --force

# Проверить, что ca в secret и caBundle в VWC совпадают
kubectl get secret ingress-nginx-admission -n ingress-nginx \
  -o jsonpath='{.data.ca}' | head -c 40
kubectl get validatingwebhookconfiguration ingress-nginx-admission \
  -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | head -c 40
# Строки должны совпасть
```

### Шаг 5: Проверить статус loadbalancer и monitoring
```bash
kubectl get application -n argocd loadbalancer monitoring \
  -o custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status
# Ожидаем: Synced / Healthy для обоих
```

### Шаг 6: Сбросить зависшие cert-manager challenges
```bash
# Посмотреть стоящие challenges
kubectl get challenge -A

# Удалить зависшие — cert-manager пересоздаст их автоматически
kubectl delete challenge -n vaultwarden --all
kubectl delete challenge -n monitoring --all

# Смотреть за прогрессом
kubectl get certificate -n vaultwarden -w
kubectl get certificate -n monitoring -w
```

### Итоговый результат

```
NAME           SYNC     HEALTH
loadbalancer   Synced   Healthy
monitoring     Synced   Healthy
```

MetalLB controller pod — 1/1 Running без restarts.  
cert-manager challenges — Ready.  
`warden.example.com` — TLS сертификат выдан.

---

## 5. Таблица изменённых файлов

| Файл | Изменение |
|------|-----------|
| `infrastructure/security/network-policies/metallb-netpol.yaml` | egress к kube-apiserver: `10.96.0.1/32:443` → `0.0.0.0/0:443,6443` (controller + speaker) |
| `apps/load-balancer.yaml` | `ignoreDifferences`: `jsonPointers[0,1]` → `jqPathExpressions: .webhooks[].clientConfig.caBundle` |
| `apps/ingress-nginx.yaml` | Добавлены `ignoreDifferences` для VWC caBundle + `retry` policy |
| `infrastructure/security/resource-limits/limit-ranges.yaml` | `min.cpu: 25m` → `10m` в `monitoring` и `vaultwarden` |

---

## 6. Уроки

### Почему `10.96.0.1/32` не работает для kube-apiserver

kube-apiserver запускается с `hostNetwork: true` на мастер-ноде. Когда под обращается к `10.96.0.1:443` (ClusterIP сервиса `kubernetes`), kube-proxy делает DNAT: пакет переписывается на `203.0.113.10:6443` (реальный IP мастера). Calico применяет NetworkPolicy **после** DNAT — правило для `10.96.0.1/32` уже не совпадает. Нужно либо `0.0.0.0/0`, либо явный ipBlock с реальным IP мастера.

### Почему `jsonPointers` для конкретных webhook индексов недостаточно

MetalLB v0.13 имеет 7 webhook-ов в одном VWC. ArgoCD игнорировал drift только для `webhooks[0]` и `webhooks[1]`. Cert-rotator обновлял caBundle в `webhooks[5]` (ipaddresspool) и `webhooks[6]` (l2advertisement) → ArgoCD видел drift → сносил caBundle → x509. Решение: `jqPathExpressions: .webhooks[].clientConfig.caBundle` покрывает все индексы.

### Почему ручная правка kubectl не работала при наличии ArgoCD selfHeal

ArgoCD с `selfHeal: true` непрерывно сравнивает live-state с git. Любое ручное изменение через `kubectl edit` или `kubectl patch` отменяется следующим циклом reconcile (обычно < 3 минут). Все изменения должны идти через git-коммит.

### Как понять что cert-manager blocker — это LimitRange, а не сеть

```bash
# Посмотреть events в namespace Certificate:
kubectl get events -n vaultwarden --sort-by='.lastTimestamp' | grep -i "forbidden\|challenge\|solver"
# Если видим "minimum cpu usage per Container is 25m, but request is 10m" — это LimitRange.
# Если видим "connection refused" или "timeout" — это сетевая политика.
```

---

## 7. Полезные команды диагностики webhook TLS

```bash
# Проверить caBundle в VWC (первые 40 символов — для сравнения с cert):
kubectl get validatingwebhookconfiguration metallb-webhook-configuration \
  -o jsonpath='{.webhooks[5].clientConfig.caBundle}' | head -c 40

# Проверить текущий cert в secret:
kubectl get secret webhook-server-cert -n metallb-system \
  -o jsonpath='{.data.ca\.crt}' | head -c 40

# Если строки не совпадают — caBundle устарел и webhook-обращения упадут с x509.

# Посмотреть cert-rotator в реальном времени:
kubectl logs -n metallb-system -l component=controller -f \
  | grep -E "cert-rotation|error|timeout|Ensuring"

# Принудительно пересоздать ingress-nginx cert:
kubectl delete secret ingress-nginx-admission -n ingress-nginx
kubectl delete job -n ingress-nginx ingress-nginx-admission-create \
  ingress-nginx-admission-patch --ignore-not-found
argocd app sync ingressnginx --force
```

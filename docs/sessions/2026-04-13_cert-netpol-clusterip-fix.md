# Устранение cert-manager-default-deny и фиксация ClusterIP ingress-nginx

**Дата:** 13.04.2026  
**Кластер:** k8master + k8worker2  
**Контекст:** `vaultwarden-tls-cert` находился в состоянии `READY: False` более 2 суток и 8 часов — Certificate был создан, но ни один Challenge никогда не завершался успехом. Сертификаты для `argocd.example.com` и `vault.example.com` были выданы раньше и ещё действовали. В ходе диагностики вскрылась цепочка из трёх независимых проблем: NetworkPolicy `cert-manager-default-deny` блокировала HTTP-01 self-check, а при попытке форс-синка ClusterIP ingress-nginx сменился — CoreDNS с захардкоженным старым адресом перестал работать.

---

## 1. Симптомы

### Сертификат застрял в False более 2d8h

```
NAME                  READY   SECRET                AGE
vaultwarden-tls-cert  False   vaultwarden-tls-cert  2d8h
```

### cert-manager логи: context deadline exceeded

```
E0413 controller/challenges.go "Accepting challenge" err=
  "failed to perform self check GET request
  'http://warden.example.com/.well-known/acme-challenge/TOKEN':
  Get \"http://warden.example.com/.well-known/acme-challenge/TOKEN\":
  context deadline exceeded"
```

Self-check — это HTTP GET-запрос, который cert-manager делает **изнутри кластера** перед тем, как сообщить Let's Encrypt «иди проверяй». Пока self-check не проходит — Let's Encrypt не контактируется вообще.

### После force-sync: новый ClusterIP — старый CoreDNS

После принудительного пересоздания ресурсов через `argocd app sync --force --replace` сервис `ingress-nginx-controller` получил новый ClusterIP (`10.107.200.146` вместо `10.96.184.160`). CoreDNS имел в Corefile захардкоженный старый адрес → DNS резолюция `*.example.com` возвращала несуществующий IP → `dial tcp 10.96.184.160:80: i/o timeout`.

### CoreDNS rewrite plugin вызвал SERVFAIL

При попытке заменить `hosts`-блок на `rewrite stop name regex` появилась ошибка:

```
nslookup warden.example.com
;; connection timed out; no servers could be reached
server can't find warden.example.com: SERVFAIL
```

---

## 2. Корневые причины

### Причина 1: cert-manager-default-deny блокировал HTTP-01 self-check

**Файл:** `infrastructure/security/network-policies/cert-manager-netpol.yaml`  
**Коммит, добавивший проблему:** `63f2269`

В коммите `63f2269` был добавлен `NetworkPolicy cert-manager-default-deny`, блокирующий весь egress из cert-manager namespace. Следом был добавлен `cert-manager-allow` с правилом `ipBlock: 10.96.0.0/12:80`, чтобы разрешить обращения к ingress-nginx ClusterIP.

Проблема: Calico применяет NetworkPolicy-правила на veth-интерфейсе пода **до DNAT**, но при обращении к ClusterIP kube-proxy делает DNAT на стороне destination. Calico и kube-proxy работают на разных уровнях стека, и для некоторых комбинаций `ipBlock` + ClusterIP трафик молча отбрасывается, несмотря на наличие разрешающего правила.

Сертификаты `argocd` (66d) и `vault` (31d) были выданы **до** добавления `cert-manager-default-deny` → они работали нормально. `vaultwarden-tls-cert` был создан **после** — и ни разу не прошёл self-check.

cert-manager имеет права `cluster-admin`. Ограничение его egress не даёт никакой реальной безопасности и при этом надёжно ломает выдачу сертификатов.

### Причина 2: ClusterIP сервиса ingress-nginx менялся при force-sync

**Файл:** `infrastructure/ingress-nginx/install-nginx.yaml`

При `argocd app sync --force --replace` Kubernetes удаляет и пересоздаёт Service. Без явно указанного `clusterIP` в spec Kubernetes присваивает новый адрес из диапазона `10.96.0.0/12`. CoreDNS `hosts`-блок имел захардкоженный старый IP → все DNS-резолюции внутри кластера возвращали несуществующий адрес.

### Причина 3: RespectIgnoreDifferences=true отсутствовал в ArgoCD apps

**Файлы:** `apps/ingress-nginx.yaml`, `apps/load-balancer.yaml`

`ignoreDifferences` без `RespectIgnoreDifferences=true` работает только для отображения дрифта — ArgoCD всё равно применяет манифест из git при каждом sync, затирая caBundle в `ValidatingWebhookConfiguration`. ingress-nginx admission webhook становился нерабочим → создание Ingress-ресурсов для ACME challenge падало с x509 ошибкой → cert-manager не мог создать Challenge Ingress.

### Причина 4: CoreDNS rewrite plugin — SERVFAIL при cross-zone резолюции

При попытке использовать `rewrite stop name regex ^(.+)\.example\.com$ ingress-nginx-controller.ingress-nginx.svc.cluster.local` в server block `example.com:53` возникал SERVFAIL. Причина: плагин `kubernetes` в блоке `example.com:53` не может резолвить `*.cluster.local` имена — это другая зона. Переписанное имя уходило обратно в тот же server block, не находило ответа и возвращало SERVFAIL. Метод отвергнут.

### Причина 5: vault namespace — LimitRange min.cpu: 50m (проактивная)

**Файл:** `infrastructure/security/resource-limits/limit-ranges.yaml`

vault LimitRange имел `min.cpu: 50m`. cert-manager ACME HTTP01 solver запрашивает `cpu: 10m`. При следующем renewal сертификата `vault.example.com` (~через 30 дней) solver pod был бы отклонён с ошибкой `forbidden: minimum cpu usage per Container is 50m, but request is 10m`. Исправлено проактивно.

---

## 3. Исправления в git

### 3.1 RespectIgnoreDifferences=true для ingress-nginx и MetalLB (коммит `f742868`)

**Файлы:** `apps/ingress-nginx.yaml`, `apps/load-balancer.yaml`

```yaml
# apps/ingress-nginx.yaml и apps/load-balancer.yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
    - RespectIgnoreDifferences=true   # добавлено
```

Без этого флага ArgoCD при каждом sync-цикле перезаписывал `caBundle` в VWC значением из git (пустым). С флагом — поля, перечисленные в `ignoreDifferences`, не трогаются при apply.

### 3.2 Понижение min.cpu в vault namespace (коммит `fae578c`)

**Файл:** `infrastructure/security/resource-limits/limit-ranges.yaml`

```yaml
# LimitRange для vault
min:
  # cert-manager ACME HTTP01 solver создаёт поды с cpu: 10m.
  # min не может быть выше 10m, иначе solver pod отклоняется LimitRange
  # и Let's Encrypt сертификаты для vault.example.com не обновляются.
  cpu: 10m      # было: 50m
  memory: 64Mi
```

### 3.3 Удаление cert-manager NetworkPolicies + фиксация ClusterIP (коммит `5fa47d1`)

**Файл:** `infrastructure/security/network-policies/cert-manager-netpol.yaml` — очищен, заменён на комментарий:

```yaml
# cert-manager — доверенный системный компонент кластера.
# default-deny не применяется: cert-manager требует широкого доступа
# (kube-apiserver, Let's Encrypt ACME, ingress-nginx ClusterIP для HTTP-01 self-check).
# Ограничение egress у cert-manager ломает выдачу сертификатов без реальной
# выгоды в безопасности — cert-manager уже имеет права cluster-admin.
```

**Файл:** `infrastructure/security/kustomization.yaml` — удалена запись `cert-manager-netpol.yaml`:

```yaml
# Было:
resources:
  - network-policies/cert-manager-netpol.yaml   # <-- удалено
  - network-policies/app-netpol.yaml
  ...
```

После удаления из kustomization.yaml ArgoCD с `prune: true` автоматически удаляет NetworkPolicy из кластера при следующем sync.

**Файл:** `infrastructure/ingress-nginx/install-nginx.yaml` — зафиксирован ClusterIP:

```yaml
# В spec сервиса ingress-nginx-controller:
spec:
  # Зафиксированный ClusterIP — не меняется при пересоздании Service.
  # Необходимо для стабильности CoreDNS split-horizon и cert-manager self-check.
  clusterIP: 10.107.200.146
  externalTrafficPolicy: Cluster
```

Kubernetes при пересоздании Service с явным `clusterIP` всегда назначает тот же адрес. Это делает CoreDNS `hosts`-блок устойчивым к force-sync.

---

## 4. Ручные команды

### Шаг 1: Обновить CoreDNS hosts-блок после смены ClusterIP

После того как force-sync изменил ClusterIP с `10.96.184.160` на `10.107.200.146`, обновил CoreDNS вручную:

```bash
# Получить текущий Corefile
kubectl get cm coredns -n kube-system -o jsonpath='{.data.Corefile}'

# Обновить IP в hosts-блоке (заменить старый IP на новый)
kubectl edit cm coredns -n kube-system
# В блоке example.com:53 → hosts {}:
# 10.107.200.146 warden.example.com
# 10.107.200.146 argocd.example.com
# 10.107.200.146 vault.example.com
# 10.107.200.146 grafana.example.com
# 10.107.200.146 prometheus.example.com

# Перезапустить CoreDNS для применения изменений
kubectl rollout restart deployment/coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system --timeout=60s
```

### Шаг 2: Синхронизировать security app для прунинга cert-manager NetworkPolicies

```bash
# Принудительный sync — ArgoCD удалит NetworkPolicy из кластера (prune: true)
argocd app sync security

# Проверить, что NetworkPolicy удалена
kubectl get networkpolicy -n cert-manager
# Ожидаем: No resources found in cert-manager namespace.
```

### Шаг 3: Форсировать переиздание сертификата

После 2d8h отказов CertificateRequest накопил экспоненциальный backoff (несколько часов между попытками). Удаление принуждает немедленную повторную попытку:

```bash
# Посмотреть текущие CertificateRequest и Challenge
kubectl get certificaterequest -n vaultwarden
kubectl get challenge -n vaultwarden

# Удалить зависший CertificateRequest — cert-manager создаст новый немедленно
kubectl delete certificaterequest -n vaultwarden --all

# Наблюдать за прогрессом
kubectl get certificate -n vaultwarden vaultwarden-tls-cert -w
# Через ~1-2 минуты: READY True
```

### Итоговый результат

```
NAME                  READY   SECRET                AGE
vaultwarden-tls-cert  True    vaultwarden-tls-cert  2d9h
```

---

## 5. Таблица изменённых файлов

| Файл | Изменение | Коммит |
|------|-----------|--------|
| `apps/ingress-nginx.yaml` | `RespectIgnoreDifferences=true` в syncOptions | `f742868` |
| `apps/load-balancer.yaml` | `RespectIgnoreDifferences=true` в syncOptions | `f742868` |
| `infrastructure/security/resource-limits/limit-ranges.yaml` | vault namespace: `min.cpu: 50m` → `10m` | `fae578c` |
| `infrastructure/security/network-policies/cert-manager-netpol.yaml` | Все NetworkPolicy удалены, файл заменён комментарием | `5fa47d1` |
| `infrastructure/security/kustomization.yaml` | Удалена запись `cert-manager-netpol.yaml` (ArgoCD prunes NP из кластера) | `5fa47d1` |
| `infrastructure/ingress-nginx/install-nginx.yaml` | Зафиксирован `clusterIP: 10.107.200.146` в spec Service | `5fa47d1` |
| **CoreDNS ConfigMap** (kube-system) | `hosts`-блок: `10.96.184.160` → `10.107.200.146` | Imperative, не git |

---

## 6. Уроки

### Почему ipBlock для ClusterIP не работает с Calico

Calico применяет NetworkPolicy на veth-интерфейсе пода. При обращении к ClusterIP kube-proxy выполняет DNAT: пакет уже имеет реальный IP пода назначения в момент, когда Calico его видит. Комбинация `ipBlock` + ClusterIP-адрес ненадёжна — трафик может молча отбрасываться. Вместо `ipBlock: 10.96.0.0/12` нужно использовать `namespaceSelector` или `podSelector`. Для системных компонентов (cert-manager, CoreDNS) — проще не ограничивать egress вообще.

### Когда NetworkPolicy не добавляет безопасности

cert-manager имеет права `cluster-admin` — может делать всё что угодно в кластере. NetworkPolicy для cert-manager ограничивала только сетевой трафик, но не API-доступ. Реальной защиты — ноль. Зато надёжно ломала выдачу TLS-сертификатов. Правило: не добавлять ограничения ради ограничений для компонентов с `cluster-admin`.

### Почему зафиксировать ClusterIP в spec — правильное решение

CoreDNS split-horizon требует знать ClusterIP ingress-nginx заранее. Без фиксации каждый `argocd app sync --force --replace` мог менять ClusterIP → CoreDNS переставал резолвить домены → cert-manager self-check падал → все TLS-сертификаты переставали продлеваться. `clusterIP: <value>` в spec Service — официально поддерживаемый Kubernetes механизм резервирования конкретного адреса.

### Почему CoreDNS rewrite с cross-zone резолюцией не работает

`rewrite stop name regex ^(.+)\.example\.com$ X.svc.cluster.local` в server block `example.com:53` вызывает SERVFAIL. Server block `example.com:53` авторитативен только для своей зоны — он не может отдать запрос плагину `kubernetes`, который работает в зоне `cluster.local`. Перезаписанное имя застревает в том же блоке и получает NXDOMAIN → SERVFAIL. Правильный подход — `hosts`-блок с явными записями и `fallthrough`.

### Почему Certificate AGE не меняется при удалении CertificateRequest

Certificate — это долгоживущий CRD-объект. Его AGE не изменяется при удалении подчинённых ресурсов (CertificateRequest, Challenge, Order). Удаление CertificateRequest только форсирует немедленную новую попытку получения сертификата — сам Certificate остаётся на месте. AGE Certificate не является индикатором свежести сертификата; используйте `kubectl describe certificate` → поле `Not After`.

### Почему ручная правка kubectl не работала при selfHeal

ArgoCD с `selfHeal: true` непрерывно сравнивает live-state с git и применяет git-манифест при обнаружении дрифта (цикл ~3 минуты). Любое `kubectl edit` или `kubectl patch` отменяется следующим reconcile. Все изменения — только через git-коммит.

---

## 7. Команды диагностики

```bash
# Проверить статус всех Certificate во всех namespace
kubectl get certificate -A

# Посмотреть причину отказа (Reason, Message)
kubectl describe certificate vaultwarden-tls-cert -n vaultwarden

# Найти активные Challenge
kubectl get challenge -A

# Смотреть cert-manager controller в реальном времени
kubectl logs -n cert-manager -l app=cert-manager -c cert-manager-controller -f \
  | grep -E "challenge|self.check|error|timeout|Certificate"

# Проверить DNS-резолюцию домена изнутри кластера
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- \
  nslookup warden.example.com
# Ожидаем ClusterIP (10.107.200.146), а не внешний IP (203.0.113.10)

# Проверить, нет ли NetworkPolicy блокирующей cert-manager egress
kubectl get networkpolicy -n cert-manager

# Проверить ClusterIP ingress-nginx (должен быть зафиксированный)
kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.clusterIP}{"\n"}'
# Ожидаем: 10.107.200.146

# Форсировать переиздание зависшего сертификата
kubectl delete certificaterequest -n <namespace> --all
kubectl delete challenge -n <namespace> --all
kubectl get certificate -n <namespace> -w
```

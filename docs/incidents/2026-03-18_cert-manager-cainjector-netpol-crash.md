# Инцидент: cert-manager-cainjector CrashLoopBackOff

**Дата:** 17-18.03.2026  
**Время начала:** ~20:50 UTC 17.03 (после рестарта мастера)  
**Время восстановления:** ~06:57 UTC 18.03  
**Продолжительность:** ~10 часов  
**Кластер:** k8master (203.0.113.10) + k8worker2 (203.0.113.20)

---

## Предыстория

После рестарта мастер ноды и восстановления kube-apiserver обнаружил что `cert-manager-cainjector` не может подняться — уходит в CrashLoopBackOff.

---

## Обнаружение

При проверке всех подов после рестарта мастера увидел:

```
cert-manager   cert-manager-cainjector-585f77dbdd-m6x8m   0/1   CrashLoopBackOff   10 (3m2s ago)
```

Количество рестартов росло — в итоге достигло 106.

---

## Диагностика — шаг 1: логи пода

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

**Причина найдена** — cainjector не может подключиться к API server через ClusterIP `10.96.0.1:443`.

---

## Диагностика — шаг 2: анализ NetworkPolicy

Проверил существующий `cert-manager-cainjector-allow`:

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 10.96.0.1/32
    ports:
      - protocol: TCP
        port: 443
```

Политика разрешает egress на `10.96.0.1:443` — выглядит правильно. Но проблема в том как Calico обрабатывает трафик.

## Причина

После DNAT пакет идёт не на `10.96.0.1:443` а на реальный IP мастер ноды `203.0.113.10:6443`. Calico применяет NetworkPolicy **после DNAT** — то есть проверяет уже изменённый адрес. Поскольку egress на `203.0.113.10:6443` не был разрешён — трафик блокировался.

Это специфика работы Calico с iptables — ClusterIP раскрывается через DNAT до проверки NetworkPolicy.

---

## Решение

Добавил в NetworkPolicy egress на реальный IP мастера для всех трёх компонентов cert-manager.

Файл `infrastructure/security/network-policies/cert-manager-netpol.yaml`:

```yaml
# Добавить во все три политики: cert-manager-allow, cert-manager-webhook-allow, cert-manager-cainjector-allow
egress:
  # ClusterIP (до DNAT)
  - to:
      - ipBlock:
          cidr: 10.96.0.1/32
    ports:
      - protocol: TCP
        port: 443
  # Реальный IP мастера (после DNAT — Calico проверяет здесь)
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
I0318 06:57:47.118726  1 reconciler.go:142] "cert-manager: Updated object" kind="validatingwebhookconfiguration"
```

Итоговый статус:

```
cert-manager-8bf7cd65c-dc5rg               1/1   Running   0
cert-manager-cainjector-585f77dbdd-x6jvx   1/1   Running   0
cert-manager-webhook-6bc56dcb94-57p6t      1/1   Running   0
```

---

## Какие ещё компоненты подвержены этой проблеме

Та же проблема может возникнуть у любого пода который обращается к API server через NetworkPolicy. После рестарта мастера проверить:

| Namespace | Компонент | Статус |
|-----------|-----------|--------|
| cert-manager | cainjector, webhook, controller | Исправлено |
| ingress-nginx | controller | Исправлено (добавлен egress на 203.0.113.10:6443) |
| keda | operator | Решается удалением пода |
| vault | agent-injector | Требует проверки |
| metallb | controller, speaker | Требует проверки |

---

## Профилактика

### После любого рестарта мастера — проверить все поды с NetworkPolicy

```bash
# Найти все CrashLoopBackOff поды
kubectl get pods -A | grep -E "CrashLoop|Error"

# Проверить логи — типичная ошибка
kubectl logs -n <namespace> <pod> | grep "i/o timeout\|connection refused\|dial tcp 10.96.0.1"

# Если ошибка про API server — удалить под, он пересоздастся
kubectl delete pod -n <namespace> <pod>
```

### Правило для NetworkPolicy с доступом к API server

Всегда указывать ОБА адреса:

```yaml
egress:
  # 1. ClusterIP kubernetes service
  - to:
      - ipBlock:
          cidr: 10.96.0.1/32
    ports:
      - protocol: TCP
        port: 443
  # 2. Реальный IP мастера (после DNAT в Calico)
  - to:
      - ipBlock:
          cidr: 203.0.113.10/32
    ports:
      - protocol: TCP
        port: 6443
```

### Почему нужны оба адреса

```
Pod → NetworkPolicy check → DNAT → kube-apiserver
         ↑                    ↑
    10.96.0.1:443      203.0.113.10:6443
    (нужно разрешить)  (нужно разрешить — Calico проверяет здесь)
```

Calico в режиме iptables применяет политики **после DNAT** — пакет уже идёт на реальный IP. Поэтому одного `10.96.0.1` недостаточно.

---

## Быстрый чеклист после рестарта мастера

```bash
# 1. Проверить все поды
kubectl get pods -A | grep -v Running | grep -v Completed

# 2. Для каждого CrashLoopBackOff — проверить логи
kubectl logs -n <ns> <pod> | grep -E "timeout|refused|dial tcp"

# 3. Если ошибка про API server — удалить под
kubectl get pods -A | grep CrashLoopBackOff \
  | awk '{print "kubectl delete pod "$2" -n "$1}' | bash

# 4. Если под продолжает падать — проверить NetworkPolicy
kubectl get networkpolicy -n <namespace> -o yaml | grep -A5 "6443\|10.96.0.1"
```

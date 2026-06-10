# ACME HTTP-01 self-check hairpin NAT — CoreDNS split-horizon fix

**Дата:** 13.04.2026  
**Затронутые сертификаты:** `vaultwarden-tls-cert` (vaultwarden), `monitoring-tls-certs` (monitoring)  
**Симптом:** Certificate `READY: False`, Challenge в состоянии `pending` навсегда.

---

## Симптомы

```
Status:
  Reason: Waiting for HTTP-01 challenge propagation: failed to perform self check
  GET request 'http://warden.example.com/.well-known/acme-challenge/...':
  dial tcp 203.0.113.10:80: i/o timeout
```

`Presented: true` — solver pod и Ingress создаются успешно.  
Самого cert'а нет — cert-manager не говорит Let's Encrypt "иди проверяй", пока self-check не пройдёт.

---

## Диагностика по шагам (что пробовали и почему не помогало)

### Шаг 1: LimitRange min.cpu блокировал solver pods
**Ошибка:** `pods "cm-acme-http-solver-..." is forbidden: minimum cpu usage per Container is 25m, but request is 10m`  
**Фикс:** Снизить `min.cpu` в LimitRange namespace'ов до 10m в git.  
**Почему ручной edit не работал:** ArgoCD selfHeal сразу возвращал значение из git (25m).

### Шаг 2: NetworkPolicy в vaultwarden/monitoring блокировала solver pod
**Ошибка:** `context deadline exceeded` — solver pod не принимал входящие соединения от ingress-nginx.  
**Фикс:** Добавить NetworkPolicy для подов с лейблом `acme.cert-manager.io/http01-solver: "true"` — ingress на порту 8089 от ingress-nginx namespace.

### Шаг 3: NetworkPolicy cert-manager блокировала порт 80
**Ошибка:** `dial tcp 203.0.113.10:80: i/o timeout`  
**Фикс:** Добавить порт 80 в egress правило `0.0.0.0/0` в cert-manager-netpol.

### Шаг 4 (КОРЕНЬ): hairpin NAT с MetalLB L2
**Ошибка:** Всё равно `i/o timeout`, несмотря на то что все netpol исправлены.  
**Причина:** MetalLB L2 использует реальные IP нод как LoadBalancer VIP (203.0.113.10, 203.0.113.20). Когда под ВНУТРИ кластера пытается достучаться до `203.0.113.10:80` — собственного IP ноды — пакет уходит через default gateway, но не возвращается обратно через kube-proxy. Это называется "hairpin NAT" и является архитектурным ограничением L2 MetalLB при использовании реальных IP нод (не отдельных VIP).  

cert-manager делает DNS-резолюцию `warden.example.com` → получает `203.0.113.10` (через CoreDNS → внешний DNS) → пытается подключиться к `203.0.113.10:80` → timeout.

---

## Решение: CoreDNS split-horizon DNS

Настроить CoreDNS так, чтобы внутри кластера домены `*.example.com` резолвились в ClusterIP сервиса `ingress-nginx-controller`, а не во внешний IP ноды. cert-manager self-check идёт на ClusterIP:80 — всегда доступен из любого пода. Снаружи кластера DNS-резолюция не меняется.

### Команды (выполнить на мастере):

```bash
# 1. Получить ClusterIP ingress-nginx
NGINX_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.clusterIP}')
echo "ingress-nginx ClusterIP: $NGINX_IP"

# 2. Бекап CoreDNS ConfigMap
kubectl get cm coredns -n kube-system -o yaml > /tmp/coredns-backup.yaml

# 3. Получить текущий Corefile и добавить split-horizon блок
CURRENT=$(kubectl get cm coredns -n kube-system -o jsonpath='{.data.Corefile}')

kubectl create configmap coredns -n kube-system --dry-run=client \
  --from-literal="Corefile=${CURRENT}
example.com:53 {
    hosts {
        ${NGINX_IP} warden.example.com
        ${NGINX_IP} argocd.example.com
        ${NGINX_IP} vault.example.com
        ${NGINX_IP} grafana.example.com
        ${NGINX_IP} prometheus.example.com
        fallthrough
    }
    forward . /etc/resolv.conf
    cache 30
}" -o yaml | kubectl apply -f -

# 4. Перезапустить CoreDNS
kubectl rollout restart deployment/coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system --timeout=60s
```

### Проверка DNS-резолюции:

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- \
  nslookup warden.example.com
# Ожидаем: ClusterIP (10.96.x.x), а не 203.0.113.10
```

### После применения:

```bash
kubectl delete challenge -n vaultwarden --all
kubectl get certificate -n vaultwarden vaultwarden-tls-cert -w
# Ожидаем: READY True через ~1-2 минуты
```

---

## Почему не DNS-01?

DNS-01 — более правильное долгосрочное решение (не требует порта 80, работает для wildcard cert). Требует API-токен DNS-провайдера (Cloudflare, Route53 и т.д.) и настройку в ClusterIssuer. CoreDNS split-horizon — более быстрое решение без зависимостей от внешних API.

---

## Изменения в git (коммиты этой инцидентной работы)

| Коммит | Изменение |
|--------|-----------|
| `5a6f717` | `limit-ranges.yaml`: `min.cpu: 25m` → `10m` в monitoring и vaultwarden |
| `0cc1355` | `vaultwarden-netpol`, `monitoring-netpol`, `ingress-nginx-netpol`: разрешить ACME solver pods на порту 8089 |
| `9a341b7` | `cert-manager-netpol`: добавить порт 80 в egress; `limit-ranges.yaml`: `app` namespace `min.cpu: 50m` → `10m` |
| `CoreDNS` | Imperative patch на CoreDNS ConfigMap в kube-system (не git-managed) |

---

## Профилактика

Для полной защиты от этой проблемы в будущем:

1. **Перейти на DNS-01** — не зависит от портов и hairpin NAT
2. **Выделить отдельные MetalLB VIP** — не использовать реальные IP нод, а взять отдельный диапазон (если провайдер это позволяет)
3. **Задокументировать CoreDNS патч** — восстанавливать после kubeadm upgrade через этот документ

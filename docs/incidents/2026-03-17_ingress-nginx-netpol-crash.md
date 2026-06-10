# Ingress-NGINX не запускается после перезагрузки мастер-ноды

**Дата инцидента:** 17 марта 2026 01:30 GMT+03

**Кластер:** k8master (203.0.113.10), k8worker2

**CNI:** Calico

**Ingress Controller:** ingress-nginx v1.8.1

**GitOps:** ArgoCD

---

## Предыстория

На кластере работал полный production-стек: ArgoCD, ingress-nginx, monitoring (Prometheus + Grafana + Loki), cert-manager, MetalLB, KEDA, Vault, Sealed Secrets. Всё управлялось через ArgoCD из Git-репозитория.

Недавно были добавлены **NetworkPolicy** для всех namespace (security application в ArgoCD) — это default-deny политики с точечными egress/ingress правилами для каждого компонента.

После перезагрузки мастер-ноды ingress-nginx контроллер перестал запускаться.

---

## Хронология расследования

### 1. Обнаружение проблемы

После ребута мастера проверяем статус подов:

```bash
root@k8master:~# kubectl get pods -n ingress-nginx -w
NAME                                        READY   STATUS    RESTARTS      AGE
ingress-nginx-controller-69c868b799-pcwzg   0/1     Running   4 (22s ago)   3m52s
ingress-nginx-controller-b7bc7796c-s765c    0/1     Running   4 (22s ago)   3m52s
```

**Два пода**, оба `0/1 Running` — ни один не может стать Ready. Рестарты растут.

> **Почему два пода?** Потому что ArgoCD при предыдущем синхе обновил deployment, создал новый ReplicaSet, но старый pod тоже не умирает — стратегия RollingUpdate держит оба, пока новый не станет Ready.

### 2. Первая попытка: удаление подов

```bash
root@k8master:~# kubectl delete pod -n ingress-nginx --all --force --grace-period=0
pod "ingress-nginx-controller-69c868b799-pcwzg" force deleted
pod "ingress-nginx-controller-b7bc7796c-s765c" force deleted
```

Новые поды создаются автоматически (Deployment), но проблема та же — `0/1 Running`.

### 3. Чистка старых ReplicaSet

```bash
root@k8master:~# kubectl get replicaset -n ingress-nginx
NAME                                  DESIRED   CURRENT   READY   AGE
ingress-nginx-controller-5bdb595b6d   0         0         0       39d
ingress-nginx-controller-69c868b799   1         1         0       7m44s
ingress-nginx-controller-7d5c6cff84   0         0         0       43d
ingress-nginx-controller-b7bc7796c    1         1         0       7d1h
ingress-nginx-controller-cf957774c    0         0         0       7d1h
```

5 ReplicaSet! Два активных (DESIRED=1), три старых (DESIRED=0). Удаляем лишний:

```bash
root@k8master:~# kubectl delete replicaset ingress-nginx-controller-b7bc7796c -n ingress-nginx
```

Теперь один pod, но он всё равно не Ready.

### 4. Анализ логов — ключевая улика

```bash
root@k8master:~# kubectl logs ingress-nginx-controller-69c868b799-wq8t7 -n ingress-nginx
```

```
-------------------------------------------------------------------------------
NGINX Ingress controller
  Release:       v1.8.1
  Build:         dc88dce9ea5e700f3301d16f971fa17c6cfe757d
  Repository:    https://github.com/kubernetes/ingress-nginx
  nginx version: nginx/1.21.6
-------------------------------------------------------------------------------
W0317 21:27:06.191646       8 client_config.go:618] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0317 21:27:06.191952       8 main.go:209] "Creating API client" host="https://10.96.0.1:443"
```

**ВСЁ. Лог обрывается.** Контроллер пытается подключиться к API серверу по адресу `10.96.0.1:443` и зависает навсегда. Через ~60 секунд liveness probe убивает под.

### 5. Анализ Events пода

```bash
root@k8master:~# kubectl describe pod ingress-nginx-controller-69c868b799-wq8t7 -n ingress-nginx | tail -15
```

```
Events:
  Warning  Unhealthy  Liveness probe failed: Get "http://172.16.0.24:10254/healthz":
           dial tcp 172.16.0.24:10254: connect: connection refused
  Normal   Killing    Container controller failed liveness probe, will be restarted
  Warning  FailedPreStopHook  PreStopHook failed
  Warning  Unhealthy  Readiness probe failed: Get "http://172.16.0.24:10254/healthz":
           dial tcp 172.16.0.24:10254: connect: connection refused
```

Контроллер даже не запускает HTTP listener на порту 10254 — он ждёт подключения к API серверу и не может пройти инициализацию.

### 6. Проверка: API сервер доступен?

```bash
root@k8master:~# curl -sk https://203.0.113.10:6443/healthz
ok
```

API сервер работает. Проблема не в нём.

### 7. Проверка iptables DNAT

```bash
root@k8master:~# iptables -t nat -L KUBE-SERVICES -n | grep 10.96.0.1
KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  0.0.0.0/0  10.96.0.1  tcp dpt:443

root@k8master:~# iptables -t nat -L KUBE-SVC-NPX46M4PTMTKRN6Y -n
Chain KUBE-SVC-NPX46M4PTMTKRN6Y (1 references)
target                     source       destination
KUBE-SEP-ZURBCCAR2TFZLTJV  0.0.0.0/0   0.0.0.0/0   /* default/kubernetes:https -> 203.0.113.10:6443 */
```

**Ключевая находка!** ClusterIP `10.96.0.1:443` маршрутизируется через DNAT на реальный IP мастера `203.0.113.10:6443`.

### 8. Проверка CA-сертификата

```bash
root@k8master:~# openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -fingerprint | md5sum
bafb8c5d4559401ae04438f0b5628970  -

root@k8master:~# kubectl get configmap kube-root-ca.crt -n ingress-nginx -o jsonpath='{.data.ca\.crt}' | openssl x509 -noout -fingerprint | md5sum
bafb8c5d4559401ae04438f0b5628970  -
```

Сертификаты совпадают — проблема не в TLS.

### 9. Попытка обратиться к API серверу из пода

```bash
root@k8master:~# kubectl exec -n ingress-nginx ingress-nginx-controller-69c868b799-mr6p4 -- wget -qO- --no-check-certificate https://10.96.0.1:443/healthz
command terminated with exit code 137
```

**Exit code 137 = SIGKILL.** Pod был убит liveness probe во время выполнения wget. Но сам факт — wget не вернул результат, значит **сетевой запрос не прошёл**.

### 10. Гипотеза: NetworkPolicy?

```bash
root@k8master:~# kubectl get networkpolicy -n ingress-nginx
NAME                             POD-SELECTOR                           AGE
ingress-nginx-controller-allow   app.kubernetes.io/name=ingress-nginx   92s
ingress-nginx-default-deny       <none>                                 105s
```

Есть **default-deny** (блокирует весь ingress и egress) и **controller-allow** (разрешает конкретные потоки).

### 11. Попытка удалить NetworkPolicy

```bash
root@k8master:~# kubectl delete networkpolicy ingress-nginx-default-deny -n ingress-nginx
root@k8master:~# kubectl delete networkpolicy ingress-nginx-controller-allow -n ingress-nginx
```

Но ArgoCD тут же их восстанавливает (синхронизация из Git):

```bash
root@k8master:~# kubectl get networkpolicy -n ingress-nginx
NAME                             POD-SELECTOR                           AGE
ingress-nginx-controller-allow   app.kubernetes.io/name=ingress-nginx   100s
ingress-nginx-default-deny       <none>                                 113s
```

> Это правильное поведение GitOps — ArgoCD следит за desired state. Удалять вручную бесполезно, нужно менять манифест в Git.

### 12. Многочисленные попытки ArgoCD sync

```bash
root@k8master:~# kubectl annotate application ingressnginx -n argocd argocd.argoproj.io/refresh=hard --overwrite
root@k8master:~# kubectl patch application ingressnginx -n argocd -p '{"operation":{"sync":{"revision":"HEAD"}}}' --type=merge
```

Не помогает — проблема не в конфигурации ingress-nginx Deployment, а в NetworkPolicy.

### 13. Проверка ресурсов

```bash
root@k8master:~# kubectl get deployment ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.template.spec.containers[0].resources}'
{"requests":{"cpu":"100m","memory":"90Mi"}}
```

Ресурсов хватает, OOM нет:

```bash
root@k8master:~# kubectl describe pod ingress-nginx-controller-69c868b799-mr6p4 -n ingress-nginx | grep -E "OOM|Killed|Limit|Memory|CPU"
# Пусто — OOMKill не было
```

### 14. Проверка probe timing

```bash
root@k8master:~# kubectl get deployment ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.template.spec.containers[0].livenessProbe}'
{"failureThreshold":5,"httpGet":{"path":"/healthz","port":10254,"scheme":"HTTP"},
 "initialDelaySeconds":60,"periodSeconds":10,"successThreshold":1,"timeoutSeconds":1}
```

Liveness даёт 60 + (5 * 10) = 110 секунд. Достаточно, если бы API сервер был доступен.

### 15. Kubelet тоже жалуется

```bash
root@k8master:~# systemctl status kubelet
```

```
E0317 21:50:08.328917 prober_manager.go:197] "Startup probe already exists"
E0317 21:50:14.327673 prober_manager.go:209] "Readiness probe already exists"
```

Kubelet путается с probe'ами из-за частых рестартов.

---

## Root Cause Analysis

После всех проверок причина стала ясна. Смотрим содержимое NetworkPolicy `ingress-nginx-controller-allow`:

```yaml
# Файл: infrastructure/security/network-policies/ingress-nginx-netpol.yaml
egress:
  # DNS — OK
  - to:
      - namespaceSelector: {}
        podSelector:
          matchLabels:
            k8s-app: kube-dns
    ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53

  # Backend namespaces — OK
  # ... app, monitoring, argocd, vault ...

  # kube-apiserver — ПРОБЛЕМА ТУТ:
  - to:
      - ipBlock:
          cidr: 10.96.0.1/32     # <-- ClusterIP
    ports:
      - protocol: TCP
        port: 443
```

### Почему ClusterIP не работает в egress NetworkPolicy с Calico

**Calico проверяет NetworkPolicy ПОСЛЕ того, как kube-proxy выполнит DNAT.**

Вот что происходит шаг за шагом:

```
1. Pod ingress-nginx отправляет пакет на 10.96.0.1:443
2. kube-proxy (iptables) выполняет DNAT:
   destination меняется на 203.0.113.10:6443
3. Пакет попадает в Calico для проверки NetworkPolicy
4. Calico видит destination = 203.0.113.10:6443
5. Проверяет egress правила:
   - 10.96.0.1/32:443? НЕТ (destination уже 203.0.113.10:6443)
6. Пакет ДРОПАЕТСЯ
```

```
Pod ──► 10.96.0.1:443 ──► [kube-proxy DNAT] ──► 203.0.113.10:6443 ──► [Calico] ──► DROP!
                                                                         │
                                                    Правило: 10.96.0.1/32:443
                                                    Пакет:   203.0.113.10:6443
                                                    Результат: НЕ СОВПАЛО → DROP
```

> **Важно:** Это особенность **Calico**. Другие CNI (например, Cilium) могут проверять NetworkPolicy ДО DNAT. С Calico нужно всегда учитывать реальный IP endpoint'а.

---

## Решение

### Что было исправлено

В файле `infrastructure/security/network-policies/ingress-nginx-netpol.yaml` добавлено egress правило с реальным IP мастера:

```yaml
    # kube-apiserver (для ingress status updates)
    # ClusterIP kubernetes service
    - to:
        - ipBlock:
            cidr: 10.96.0.1/32
      ports:
        - protocol: TCP
          port: 443
    # Реальный IP мастер-ноды (после DNAT Calico проверяет реальный адрес)
    - to:
        - ipBlock:
            cidr: 203.0.113.10/32
      ports:
        - protocol: TCP
          port: 6443
```

### Дополнительные улучшения

**1. Добавлен `startupProbe`** в `infrastructure/ingress-nginx/install-nginx.yaml`:

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 10254
    scheme: HTTP
  failureThreshold: 30    # 30 попыток * 5 сек = 150 сек на старт
  periodSeconds: 5
  timeoutSeconds: 2
```

Зачем: даёт контейнеру до 2.5 минут на инициализацию. Пока startupProbe не пройдёт, livenessProbe и readinessProbe не запускаются — контроллер не будет убит во время медленного старта.

**2. Уменьшен `revisionHistoryLimit`** с 10 до 2:

```yaml
spec:
  revisionHistoryLimit: 2
```

Зачем: предотвращает накопление десятков старых ReplicaSet.

### Как применить

```bash
# 1. Запушить изменения в Git (ArgoCD подхватит)
# 2. Или принудительно синхронизировать:
kubectl annotate application security -n argocd argocd.argoproj.io/refresh=hard --overwrite
kubectl annotate application ingressnginx -n argocd argocd.argoproj.io/refresh=hard --overwrite

# 3. Рестартнуть ingress-nginx
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx

# 4. Дождаться Ready
kubectl get pods -n ingress-nginx -w
# Ожидаемый результат: 1/1 Running

# 5. Почистить старые ReplicaSet (необязательно)
kubectl get rs -n ingress-nginx -o jsonpath='{range .items[?(@.spec.replicas==0)]}{.metadata.name}{"\n"}{end}' | \
  xargs -I{} kubectl delete rs {} -n ingress-nginx
```

---

## Теория: Kubernetes Probes (Startup / Liveness / Readiness)

### Три типа проб

| Probe | Когда запускается | Что делает при фейле | Назначение |
|-------|------------------|---------------------|-----------|
| **startupProbe** | Сразу при старте контейнера | Рестартит контейнер (после исчерпания попыток) | Дать время на первый запуск |
| **livenessProbe** | После прохождения startupProbe | Рестартит контейнер | Детектить зависание |
| **readinessProbe** | После прохождения startupProbe | Убирает pod из Service endpoints | Не слать трафик в неготовый pod |

### Порядок работы

```
Контейнер стартует
    │
    ▼
[startupProbe] ── проверяет каждые periodSeconds
    │                  │
    │ (пройдёт)        │ (не пройдёт за failureThreshold * periodSeconds)
    ▼                  ▼
[livenessProbe]    KILL → Restart
[readinessProbe]
    │
    ▼
Pod Ready, трафик идёт
```

### Без startupProbe (как было)

```
T=0:   Контейнер стартует
T=10:  livenessProbe начинает проверять (initialDelaySeconds=10)
T=10:  Контейнер ещё подключается к API серверу...
T=20:  liveness FAIL #1
T=30:  liveness FAIL #2
T=40:  liveness FAIL #3
T=50:  liveness FAIL #4
T=60:  liveness FAIL #5 → KILL → Restart → CrashLoopBackOff
```

### С startupProbe (как стало)

```
T=0:    Контейнер стартует
T=0:    startupProbe начинает проверять
T=5:    startup FAIL #1 (ещё подключается к API)
T=10:   startup FAIL #2
...
T=60:   startup PASS! Контроллер запустился
T=60:   Включаются livenessProbe и readinessProbe
T=70:   liveness PASS, readiness PASS
T=70:   Pod становится Ready (1/1)
```

Максимальное время на старт: `30 * 5 = 150 секунд (2.5 минуты)`.

---

## Выводы и правила на будущее

### 1. NetworkPolicy с Calico: всегда добавлять реальный IP
При написании egress-правил к kube-apiserver нужно **два правила**:
- ClusterIP: `10.96.0.1/32:443`
- Реальный IP мастера: `<MASTER_IP>/32:6443`

### 2. Тестировать NetworkPolicy перед коммитом
```bash
# Временно убрать deny и проверить
kubectl delete networkpolicy <deny-policy> -n <namespace>
# Проверить что pod работает
# Применить policy обратно и проверить снова
```

### 3. Всегда использовать startupProbe
Для любого контейнера, который при старте подключается к внешним сервисам (API server, database, message queue), добавлять startupProbe с щедрым таймаутом.

### 4. Не удалять NetworkPolicy вручную при GitOps
ArgoCD восстановит их. Нужно менять манифесты в Git.

### 5. Держать revisionHistoryLimit маленьким
2 вместо дефолтных 10. Уменьшает мусор в виде старых ReplicaSet.

---

## Полезные команды для подобных ситуаций

```bash
# Полная диагностика пода
kubectl describe pod <pod> -n <ns> | grep -E "Ready|Started|Liveness|Readiness|Warning|OOM|Killed"

# Логи предыдущего крашнутого контейнера
kubectl logs <pod> -n <ns> --previous

# Проверить egress из пода к API серверу
kubectl exec -n <ns> <pod> -- wget -qO- --no-check-certificate https://10.96.0.1:443/healthz

# Увидеть DNAT правила для kubernetes service
iptables -t nat -L KUBE-SERVICES -n | grep 10.96.0.1

# Сравнить CA-сертификаты
openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -fingerprint | md5sum
kubectl get configmap kube-root-ca.crt -n <ns> -o jsonpath='{.data.ca\.crt}' | openssl x509 -noout -fingerprint | md5sum

# Проверить все NetworkPolicy в namespace
kubectl get networkpolicy -n <ns> -o yaml
```

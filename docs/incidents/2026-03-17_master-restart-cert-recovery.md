# Инцидент: рестарт мастер ноды k8master

**Дата:** 17.03.2026  
**Время начала:** ~20:28 UTC  
**Время восстановления:** ~20:49 UTC  
**Продолжительность:** ~21 минута  
**Кластер:** k8master (203.0.113.10) + k8worker2 (203.0.113.20)

---

## Предыстория

Потребовался плановый рестарт мастер ноды. Перед рестартом сделал снапшот etcd:

```bash
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup-20260317.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Вывод подтвердил валидность снапшота:
```
Snapshot saved at /tmp/etcd-backup-20260317.db
fd108d61, 9998086, 2298, 32 MB
```

На воркере запустил мониторинг доступности API server:

```bash
while true; do
  curl -sk https://203.0.113.10:6443/healthz && echo " OK $(date)" || echo " FAIL $(date)"
  sleep 2
done
```

---

## Инцидент 1 — kube-apiserver не поднялся

### Обнаружение

Через ~1 минуту после рестарта получил алерт от Alertmanager в Telegram:

```
🔥 АЛАРМ: critical
Событие: InstanceDown
Проект: Инфраструктура
Объект: 172.16.0.202:9100
Описание: Node Exporter перестал отдавать метрики.
Статус: В ПРОЦЕССЕ (FIRING)
Начало: 20:29:43 17.03 UTC
```

Это node-exporter на мастере — недоступен потому что нода перезагружалась. Мониторинг отработал корректно.

Параллельно увидел что мониторинг на воркере показывает FAIL больше 7 минут — это уже явно проблема, а не просто медленный старт:

```
FAIL Tue Mar 17 08:28:48 PM UTC 2026
FAIL Tue Mar 17 08:28:50 PM UTC 2026
...
FAIL Tue Mar 17 08:37:53 PM UTC 2026
FAIL Tue Mar 17 08:38:53 PM UTC 2026
```

### Диагностика — шаг 1: статус kubelet

Зашёл по ssh и проверил kubelet:

```bash
systemctl status kubelet
```

Вывод:
```
● kubelet.service - kubelet: The Kubernetes Node Agent
     Active: active (running) since Tue 2026-03-17 20:28:29 UTC

Mar 17 20:40:52 k8master kubelet[1106]: E "Unable to register node with API server"
Mar 17 20:40:52 k8master kubelet[1106]: E "Failed to get Node status to reconstruct device paths"
                                          err="dial tcp 203.0.113.10:6443: connect: connection refused"
```

Kubelet запущен, но не может достучаться до API server.

### Диагностика — шаг 2: статус static pods

```bash
crictl ps -a | grep apiserver
```

Вывод:
```
fc342e4f06fd3  5c6acd67e9cd1  About a minute ago  Exited  kube-apiserver  7  kube-apiserver-k8master
```

Контейнер в статусе `Exited` с 7 рестартами — CrashLoopBackOff.

### Диагностика — шаг 3: логи apiserver

```bash
crictl logs fc342e4f06fd3 2>&1 | tail -30
```

Ключевая ошибка:
```
E0317 20:39:35.224088  1 run.go:72] "command failed" 
err="open /etc/kubernetes/pki/apiserver.crt: no such file or directory"
```

**Причина найдена** — отсутствует сертификат `apiserver.crt`.

### Диагностика — шаг 4: проверка PKI

```bash
ls /etc/kubernetes/pki/
```

Вывод:
```
apiserver-etcd-client.crt     apiserver-kubelet-client.key  ca.key  front-proxy-ca.crt  sa.pub
apiserver-etcd-client.key     backup                        ca.srl  front-proxy-ca.key
apiserver-kubelet-client.crt  ca.crt                        etcd    sa.key
```

Отсутствуют:
- `apiserver.crt` / `apiserver.key`
- `front-proxy-client.crt` / `front-proxy-client.key`

### Решение — восстановление сертификатов

```bash
kubeadm init phase certs all
```

Вывод:
```
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8master kubernetes 
kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] 
and IPs [10.96.0.1 203.0.113.10]
```

Проверил что сертификаты появились:

```bash
ls -la /etc/kubernetes/pki/apiserver*
```

Вывод:
```
-rw-r--r-- 1 root root 1285 Mar 17 20:42 /etc/kubernetes/pki/apiserver.crt
-rw------- 1 root root 1679 Mar 17 20:42 /etc/kubernetes/pki/apiserver.key
-rw-r--r-- 1 root root 1123 Jan  8 15:28 /etc/kubernetes/pki/apiserver-etcd-client.crt
-rw------- 1 root root 1679 Jan  8 15:28 /etc/kubernetes/pki/apiserver-etcd-client.key
```

Сертификаты появились, но apiserver всё равно не поднялся — kubelet не пересоздал pod автоматически. Принудительно перезапустил kubelet:

```bash
systemctl restart kubelet
```

Через 30 секунд мониторинг на воркере:

```
FAIL Tue Mar 17 08:49:24 PM UTC 2026
 OK  Tue Mar 17 08:49:54 PM UTC 2026
```

**API server восстановлен.**

---

## Инцидент 2 — CrashLoopBackOff после восстановления

### Обнаружение

После восстановления API server проверил все поды:

```bash
kubectl get pods -A
```

Обнаружил два пода в CrashLoopBackOff:

```
NAMESPACE      NAME                                       READY   STATUS             RESTARTS
cert-manager   cert-manager-cainjector-585f77dbdd-m6x8m  0/1     CrashLoopBackOff   10 (3m2s ago)
keda           keda-operator-f75fb57f8-b8kdq              0/1     CrashLoopBackOff   10 (2m40s ago)
```

### Диагностика

```bash
kubectl logs cert-manager-cainjector-585f77dbdd-m6x8m -n cert-manager --previous | tail -5
```

Вывод:
```
E0317 20:48:59 main.go:45] "cert-manager: error executing command" 
err="unable to retrieve the complete list of server APIs: 
Get \"https://10.96.0.1:443/apis/apiextensions.k8s.io/v1\": 
dial tcp 10.96.0.1:443: i/o timeout"
```

```bash
kubectl logs keda-operator-f75fb57f8-b8kdq -n keda --previous | tail -5
```

Вывод:
```
2026-03-17T20:49:21Z ERROR scaleclient not able to get Kubernetes version
{"error": "Get \"https://10.96.0.1:443/version?timeout=32s\": 
dial tcp 10.96.0.1:443: connect: connection refused"}

2026-03-17T20:49:21Z ERROR setup unable to init scale client
```

**Причина:** оба пода стартовали пока API server ещё не был готов. Не смогли подключиться к ClusterIP `10.96.0.1:443` и ушли в exponential backoff до 5 минут — сами не восстановятся.

Проверил что ClusterIP сейчас доступен:

```bash
curl -sk https://10.96.0.1:443/healthz
```

Вывод:
```
ok
```

API server работает — поды просто застряли в backoff цикле.

### Решение

```bash
kubectl delete pod cert-manager-cainjector-585f77dbdd-m6x8m -n cert-manager
kubectl delete pod keda-operator-f75fb57f8-b8kdq -n keda
```

Поды пересоздались и успешно подключились к API server. Кластер восстановлен.

---

## Итоговое состояние

```bash
kubectl get pods -A | grep -v Running
```

Все поды в статусе Running. Кластер полностью восстановлен в 20:49 UTC.

---

## Профилактика

### Скрипт бэкапа — настроить один раз

```bash
cat > /usr/local/bin/k8s-backup.sh << 'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d-%H%M)
mkdir -p /root/backups

# etcd snapshot
ETCDCTL_API=3 etcdctl snapshot save /root/backups/etcd-$DATE.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# PKI backup
cp -r /etc/kubernetes/pki/ /root/backups/pki-$DATE/

# Хранить только 7 последних бэкапов
ls -t /root/backups/etcd-*.db | tail -n +8 | xargs rm -f 2>/dev/null
ls -dt /root/backups/pki-*/ | tail -n +8 | xargs rm -rf 2>/dev/null

echo "Backup completed: $DATE"
EOF
chmod +x /usr/local/bin/k8s-backup.sh

# Автоматически каждую ночь в 3:00
echo "0 3 * * * root /usr/local/bin/k8s-backup.sh >> /var/log/k8s-backup.log 2>&1" \
  > /etc/cron.d/k8s-backup
```

### Чеклист перед рестартом мастера

```bash
# 1. Бэкап
/usr/local/bin/k8s-backup.sh

# 2. Проверить валидность
ETCDCTL_API=3 etcdctl snapshot status /root/backups/etcd-$(date +%Y%m%d)*.db

# 3. Мониторинг на воркере в отдельном терминале
while true; do
  curl -sk https://203.0.113.10:6443/healthz && echo " OK $(date)" || echo " FAIL $(date)"
  sleep 5
done

# 4. Рестарт
reboot
```

### Алгоритм восстановления после рестарта

```bash
# Если API server не поднялся через 3 минуты:

# 1. Проверить статус apiserver
crictl ps -a | grep apiserver

# 2. Посмотреть логи
crictl logs $(crictl ps -a | grep apiserver | awk '{print $1}') 2>&1 | tail -30

# 3. Если ошибка про сертификаты
kubeadm init phase certs all
systemctl restart kubelet

# 4. После восстановления — найти CrashLoopBackOff поды
kubectl get pods -A | grep CrashLoopBackOff

# 5. Перезапустить их
kubectl get pods -A | grep CrashLoopBackOff \
  | awk '{print "kubectl delete pod "$2" -n "$1}' | bash
```

---

## Сводная таблица

| Проблема | Причина | Решение | Профилактика |
|---------|---------|---------|-------------|
| apiserver не стартует | Пропали сертификаты PKI | `kubeadm init phase certs all` + `systemctl restart kubelet` | Бэкап PKI перед рестартом |
| CrashLoopBackOff подов | Race condition — поды стартовали до API server | `kubectl delete pod` для каждого | Скрипт проверки после рестарта |

### Логическая структура документации

Если рассматривать запуск кластера с нуля на bare-metal или VPS, хронология развертывания должна строиться от базового слоя к прикладному. 
Вот как выглядит правильный флоу и почему:

1. **Инициализация и сеть (CNI):** Кластер не живет без сети между нодами.
2. **Безопасность (RBAC):** Доступы настраиваются до того, как в кластере появится важная инфраструктура.
3. **Маршрутизация трафика:** LoadBalancer (MetalLB) и Ingress-контроллер — фундамент для доступа извне.
4. **Управление секретами и SSL:** Секреты для Registry и cert-manager нужны до деплоя приложений и мониторинга, чтобы сразу получать валидные сертификаты.
5. **Мониторинг:** Разворачивается на готовую инфраструктуру с Ingress и SSL.
6. **Приложения:** Деплоятся в самом конце, используя все готовые системные сервисы.

Ниже представлены переработанные файлы инструкций.

#### Файл: `Инициализация кластера и базовая сеть.md`

> Этот этап поднимает control-plane, подключает worker-узлы и обеспечивает сетевую связность между подами через Calico.

# Инициализация кластера и настройка сети

## 1. Запуск кластера через Ansible
Вся информация по установке нод находится в директории `new_nodes/ubn24_containerd_kub1.35`.

Запуск настройки узлов:
```bash
ansible-playbook k8s-node.yml -i k8worker2, -e "node_hostname=k8worker2 k8s_version=1.35"

```

После завершения настройки серверов, инициализируем control-plane на master-ноде:

```bash
kubeadm init --apiserver-advertise-address=203.0.113.10

```

Далее необходимо подключить worker-ноды командой `kubeadm join`, которую выдаст предыдущая команда.

## 2. Установка CNI (Calico)

Для маршрутизации трафика между подами необходим сетевой плагин.

Применение манифеста Calico:

```bash
kubectl apply -f [https://docs.projectcalico.org/manifests/calico.yaml](https://docs.projectcalico.org/manifests/calico.yaml)

```

## 3. Проверка работоспособности

Убеждаемся, что ноды готовы, а системные компоненты запущены:

```bash
kubectl get nodes
kubectl get pods -n kube-system | grep calico

```

Ожидаемый результат: все ноды в статусе `Ready`, поды Calico, `coredns`, `etcd` и `apiserver` в статусе `Running`.


#### Файл: `Настройка RBAC и доступов.md`

> Доступ в кластер настраивается через Bastion-хост с использованием отдельного пользователя без admin-прав, используя Role и RoleBinding для гранулярного контроля.

# Настройка доступов через Bastion (RBAC)

Архитектура безопасности подразумевает использование отдельного пользователя `bastion-user`, аутентификацию по клиентскому сертификату и ограничение прав только в namespace `default`.

## 1. Генерация ключей и сертификатов (на Master)
```bash
mkdir ~/rbac && cd ~/rbac

# Генерация ключа и CSR
openssl genrsa -out bastion-user.key 2048
openssl req -new -key bastion-user.key -out bastion-user.csr -subj "/CN=bastion-user/O=bastion"

# Подпись сертификата через CA кластера
openssl x509 -req -in bastion-user.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out bastion-user.crt -days 365
```

## 2. Настройка Role и RoleBinding

Создаем файл `rbac-bastion.yml` с правами только на чтение (`get`, `list`, `watch`) для подов, сервисов и деплойментов:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: bastion-readonly
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bastion-readonly-binding
  namespace: default
subjects:
- kind: User
  name: bastion-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: bastion-readonly
  apiGroup: rbac.authorization.k8s.io

```

Применяем: `kubectl apply -f rbac-bastion.yml`

## 3. Настройка kubeconfig

Генерируем конфиг для пользователя:

```bash
kubectl config set-credentials bastion-user --client-certificate=bastion-user.crt --client-key=bastion-user.key --embed-certs=true
kubectl config set-context bastion --cluster=kubernetes --user=bastion-user --namespace=default
kubectl config view --context=bastion --minify --flatten > bastion.kubeconfig

```

Копируем конфиг на Bastion-хост (например, `k8worker1`):

```bash
# На Bastion-хосте предварительно: mkdir -p /root/.kube
scp bastion.kubeconfig root@k8worker1:/root/.kube/config

```

#### Файл: `Балансировка нагрузки и Ingress.md`
> Для предоставления доступа извне на VPS используется MetalLB, который эмулирует облачный LoadBalancer и передает внешний IP Ingress-контроллеру NGINX.


# Настройка маршрутизации: MetalLB + NGINX Ingress

## 1. Установка MetalLB
MetalLB необходим для выделения внешних IP-адресов сервисам типа `LoadBalancer`.
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml

```

Настройка пула адресов (используем реальный IP ноды/VPS):

```yaml
# metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: vps-pool
  namespace: metallb-system
spec:
  addresses:
    - 203.0.113.10/32
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - vps-pool

```

`kubectl apply -f metallb-config.yaml`

## 2. Установка NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace

```

Проверка выделения внешнего IP:

```bash
kubectl get svc -n ingress-nginx

```

Контроллер должен получить EXTERNAL-IP (в нашем случае `203.0.113.10`).


#### Файл: `Управление секретами и SSL.md`
> Данный этап автоматизирует получение HTTPS сертификатов от Let's Encrypt через cert-manager и настраивает секреты для скачивания приватных Docker-образов.

# Секреты инфраструктуры и SSL сертификаты

## 1. Секрет для Docker Registry (regcred)
Для скачивания образов из приватного GitLab Registry создаем секрет `regcred`.
```bash
kubectl create secret docker-registry regcred \
  --docker-server=git.example.org:5050 \
  --docker-username=k8s-image-pull \
  --docker-password=<GITLAB_PAT> \
  --docker-email=none@example.com \
  -n default
```

*Примечание: Если используется namespace `app`, секрет нужно создать и там.*

## 2. Установка cert-manager

Система для автоматического выпуска и продления SSL-сертификатов.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml

```

## 3. Настройка ClusterIssuer

Создаем глобальный ресурс для запроса сертификатов Let's Encrypt Production:

```yaml
# cluster_issuer.yml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          class: nginx

```

`kubectl apply -f cluster_issuer.yml`

Для автоматического выпуска SSL к любому Ingress достаточно добавить аннотацию: `cert-manager.io/cluster-issuer: "letsencrypt-prod"`.


#### Файл: `Развертывание стека мониторинга.md`
> Развертывание Promethues, Grafana и Alertmanager требует подготовки персистентных данных с помощью HostPath, настройки Service Discovery и маршрутизации алертов в Telegram.

# Развертывание стека мониторинга

## 1. Подготовка хранилища на хосте
Так как используется `HostPath` для БД Grafana, подготавливаем директорию на Master-ноде:
```bash
mkdir -p /mnt/grafana_data
chown -R 472:472 /mnt/grafana_data
chmod -R 775 /mnt/grafana_data
```

## 2. Развертывание компонентов

Создаем namespace и права:

```bash
kubectl create namespace monitoring
kubectl apply -f 01-rbac.yml # Из исходных манифестов
```

Генерация ConfigMap для правил и шаблонов:

```bash
kubectl create configmap alertmanager-templates -n monitoring --from-file=rules/telegram_alert.tmpl
kubectl create configmap prometheus-rules -n monitoring --from-file=rules/availability.yml --from-file=rules/app_monitoring.yml

```

Запуск сервисов:

```bash
kubectl apply -f 02-config.yml          # Config Prometheus
kubectl apply -f 06-alertmanager.yml    # Alertmanager
kubectl apply -f 04-node-exporter.yml   # Node Exporter DaemonSet
kubectl apply -f 03-main-stack.yml      # Prometheus + Grafana (HostPath)
kubectl apply -f 05-ingress-graf-prom.yml # Ingress
```

## 3. Особенности архитектуры

* **HostPath & NodeSelector:** Grafana жестко привязана к `k8master` и использует локальную директорию сервера во избежание потери данных.
* **Service Discovery:** Prometheus автоматически находит таргеты сбора через `kubernetes_sd_configs`.


#### Файл: `Развертывание тестового приложения.md`
> Финальная стадия демонстрирует полноценный деплой микросервиса, использующего ранее созданный imagePullSecrets, сервис ClusterIP и Ingress для приема трафика из интернета по HTTPS.

# Развертывание тестового приложения

## 1. Deployment и Service
Сервис изолируется внутри кластера (`ClusterIP`), а приложение использует подготовленный ранее секрет `regcred`.

```yaml
# app-stack.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: project-test
  namespace: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: project-test
  template:
    metadata:
      labels:
        app: project-test
    spec:
      containers:
        - name: project-test
          image: [git.example.org:5050/y.shumaylov/project-test:latest](https://git.example.org:5050/y.shumaylov/project-test:latest)
          ports:
            - containerPort: 8080
          env:
            - name: APP_PORT
              value: "8080"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
      imagePullSecrets:
        - name: regcred
---
apiVersion: v1
kind: Service
metadata:
  name: project-test
  namespace: app
spec:
  type: ClusterIP
  selector:
    app: project-test
  ports:
    - port: 8080
      targetPort: 8080

```

## 2. Ingress с поддержкой SSL

Используем Ingress-контроллер и cert-manager.

```yaml
# app-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-app
  namespace: app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - test-app.example.com
    secretName: test-app-tls-certs
  rules:
    - host: test-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: project-test
                port:
                  number: 8080

```

## 3. Проверка

Домен должен быть направлен на IP-адрес MetalLB (203.0.113.10).

```bash
curl -I https://test-app.example.com
```



# Далее идет Мониторинг.

# Полное описание мониторинга с содержимым файлов

Ниже представлено детальное описание каждого компонента в порядке их запуска. Содержимое файлов приведено в том виде, в котором оно должно быть сохранено для корректной работы стека.

---

### 1. Основа и права доступа (01-rbac.yml)

**Описание:** Создает изолированную среду `monitoring` и наделяет Prometheus правами «зрителя» кластера. Это позволяет ему опрашивать API Kubernetes для поиска новых подов и сервисов.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-viewer
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-viewer-binding
subjects:
  - kind: ServiceAccount
    name: default
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-viewer
  apiGroup: rbac.authorization.k8s.io

```

---

### 2. Конфигурация сбора метрик (02-config.yml)

**Описание:** Основной конфиг Prometheus. Здесь прописано, как часто собирать данные (15 сек), куда слать алерты (на Alertmanager) и какие поды мониторить (фильтр по неймспейсам `app` и `monitoring`).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s

    alerting:
      alertmanagers:
      - static_configs:
        - targets:
          - alertmanager.monitoring.svc.cluster.local:9093

    rule_files:
      - "/etc/prometheus/rules/*.yml"

    scrape_configs:
      - job_name: 'node-exporter'
        kubernetes_sd_configs:
          - role: endpoints
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_label_app]
            action: keep
            regex: node-exporter

      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            action: keep
            regex: (app|monitoring)
          - source_labels: [__meta_kubernetes_namespace]
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_container_name]
            target_label: container

```

---

### 3. Запуск серверов (03-main-stack.yml)

**Описание:** Деплоймент самих приложений Prometheus и Grafana.

* **Prometheus:** монтирует конфиги из ConfigMap.
* **Grafana:** использует `initContainer` для настройки прав на `/var/lib/grafana` и `nodeSelector` для привязки к мастер-ноде, где подготовлена папка для данных.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        args: ["--config.file=/etc/prometheus/prometheus.yml", "--storage.tsdb.path=/prometheus/"]
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus/
        - name: rules-volume
          mountPath: /etc/prometheus/rules/
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
      - name: rules-volume
        configMap:
          name: prometheus-rules
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      nodeSelector:
        kubernetes.io/hostname: k8master
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      initContainers:
      - name: fix-permissions
        image: busybox:latest
        command: ["sh", "-c", "chown -R 472:472 /var/lib/grafana && chmod -R 775 /var/lib/grafana"]
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        securityContext:
          runAsUser: 0
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-admin-secret
              key: admin-password
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
      securityContext:
        fsGroup: 472
        runAsUser: 472
      volumes:
      - name: grafana-storage
        hostPath:
          path: /mnt/grafana_data
          type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  selector:
    app: grafana
  ports:
  - port: 80
    targetPort: 3000

```

---

### 4. Метрики железа (04-node-exporter.yml)

**Описание:** Собирает системные метрики (CPU, RAM, Disk) со всех нод кластера. Работает как DaemonSet.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
          - containerPort: 9100
            name: http
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    app: node-exporter
  ports:
  - port: 9100
    name: http

```

---

### 5. Доступ извне (05-ingress-grafana.yml)

**Описание:** Настройка HTTPS доступа к веб-интерфейсам. Использует cert-manager для выпуска SSL.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - grafana.example.com
    - prometheus.example.com
    secretName: monitoring-tls-certs
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 80
  - host: prometheus.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus
            port:
              number: 9090

```

---

### 6. Центр уведомлений (06-alertmanager.yml)

**Описание:** Принимает сигналы о сбоях и отправляет их в Telegram.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
    templates:
    - '/etc/alertmanager/templates/*.tmpl'
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'telegram'
    receivers:
    - name: 'telegram'
      telegram_configs:
      - bot_token: '<TELEGRAM_BOT_TOKEN>'
        chat_id: <TELEGRAM_CHAT_ID>
        parse_mode: 'HTML'
        message: '{{ template "telegram.default.message" . }}'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager:latest
        args: ["--config.file=/etc/alertmanager/alertmanager.yml"]
        ports:
          - containerPort: 9093
        volumeMounts:
        - name: config-volume
          mountPath: /etc/alertmanager
        - name: templates-volume
          mountPath: /etc/alertmanager/templates
      volumes:
      - name: config-volume
        configMap:
          name: alertmanager-config
      - name: templates-volume
        configMap:
          name: alertmanager-templates
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  selector:
    app: alertmanager
  ports:
    - port: 9093
      targetPort: 9093

```

---

### 7. Правила алертов (07-prom-rules.yml)

**Описание:** Содержит описание условий, при которых срабатывает алерт (например, если сервер упал).

```yaml
apiVersion: v1
data:
  availability.yml: |
    groups:
    - name: availability
      rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Сервер {{ $labels.instance }} упал"
          description: "Node Exporter перестал отдавать метрики."
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring

```

---

### 8. Сбор логов: Loki (08-loki.yml)

**Описание:** Хранилище логов с привязкой к мастер-ноде для персистентности данных.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
  namespace: monitoring
data:
  loki.yaml: |
    auth_enabled: false
    server:
      http_listen_port: 3100
    common:
      path_prefix: /loki
      storage:
        filesystem:
          chunks_directory: /loki/chunks
          rules_directory: /loki/rules
      replication_factor: 1
      ring:
        kvstore:
          store: inmemory
    schema_config:
      configs:
        - from: 2020-10-24
          store: boltdb-shipper
          object_store: filesystem
          schema: v11
          index:
            prefix: index_
            period: 24h
    storage_config:
      boltdb_shipper:
        active_index_directory: /loki/boltdb-shipper-active
        cache_location: /loki/boltdb-shipper-cache
        shared_store: filesystem
      filesystem:
        directory: /loki/chunks
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loki
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: loki
  template:
    metadata:
      labels:
        app: loki
    spec:
      nodeSelector:
        kubernetes.io/hostname: k8master
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      initContainers:
      - name: fix-permissions
        image: busybox:latest
        command: ["sh", "-c", "chown -R 10001:10001 /loki && chmod -R 775 /loki"]
        volumeMounts:
        - name: loki-storage
          mountPath: /loki
        securityContext:
          runAsUser: 0
      containers:
      - name: loki
        image: grafana/loki:2.9.2
        args: ["-config.file=/etc/loki/loki.yaml"]
        ports:
        - containerPort: 3100
        volumeMounts:
        - name: config
          mountPath: /etc/loki
        - name: loki-storage
          mountPath: /loki
      volumes:
      - name: config
        configMap:
          name: loki-config
      - name: loki-storage
        hostPath:
          path: /mnt/loki_data
          type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  name: loki
  namespace: monitoring
spec:
  ports:
  - port: 3100
    targetPort: 3100
  selector:
    app: loki

```

---

### 9. Агент логов: Promtail (09-promtail.yml)

**Описание:** Читает логи контейнеров из `/var/log/pods` и пересылает их в Loki.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: monitoring
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
    positions:
      filename: /run/promtail/positions.yaml
    clients:
      - url: http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push
    scrape_configs:
    - job_name: kubernetes-pods-static
      static_configs:
      - targets:
          - localhost
        labels:
          job: kubernetes-pods
          __path__: /var/log/pods/*/*/*.log
      pipeline_stages:
      - docker: {}
      - regex:
          expression: "^/var/log/pods/(?P<namespace>[^_]+)_(?P<pod_name>[^_]+)_(?P<uid>[a-z0-9-]+)/(?P<container_name>[^_]+)/.*log$"
          source: filename
      - labels:
          namespace:
          pod: pod_name
          container: container_name
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: promtail
  template:
    metadata:
      labels:
        app: promtail
    spec:
      serviceAccountName: promtail-sa
      containers:
      - name: promtail
        image: grafana/promtail:2.9.2
        args: ["-config.file=/etc/promtail/promtail.yaml"]
        securityContext:
          runAsUser: 0
          privileged: true
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: run
          mountPath: /run/promtail
        - name: pods
          mountPath: /var/log/pods
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: promtail-config
      - name: run
        hostPath:
          path: /run/promtail
      - name: pods
        hostPath:
          path: /var/log/pods

```

---

### Секреты и прочее

* **grafana-sealed-secret.yaml:** Содержит зашифрованный пароль от Grafana.
* **promtail-rbac.yml:** Права для агента логов на чтение структуры кластера.
* **kustomization.yaml:** Файл-сборщик, указывающий `kubectl apply -k .` порядок и состав деплоя.

Все файлы должны находиться в одной директории (кроме правил в папке `rules/`) для корректной работы автоматического развертывания.

# папка rules для Мониторинга.

Для полноценного контроля за кластером стоит добавить мониторинг системных ресурсов (процессор, память, диск) и проверку здоровья самого стека логирования (Loki и Promtail).

Ниже представлены новые файлы и дополнения к существующим. Все они находятся в директории, которая создана в папке мониторинга с основными конфигами. Под папкой `rules`.

### File: ./system_resources.yml

```yaml
groups:
- name: system_resources
  rules:
  # Утилизация процессора более 85%
  - alert: HostHighCpuLoad
    expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Высокая нагрузка на CPU (Узел: {{ $labels.instance }})"
      description: "Загрузка процессора составляет {{ $value | printf \"%.2f\" }}%."

  # Использование оперативной памяти более 90%
  - alert: HostLowMemory
    expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 90
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Заканчивается память (Узел: {{ $labels.instance }})"
      description: "Использовано {{ $value | printf \"%.2f\" }}% оперативной памяти."

  # Свободное место на диске менее 10%
  - alert: HostDiskSpaceLow
    expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 10
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Критически мало места на диске (Узел: {{ $labels.instance }})"
      description: "На разделе / осталось менее 10% свободного места."

```

### File: ./logging_stack.yml

```yaml
groups:
- name: logging_stack
  rules:
  # Проверка доступности Loki
  - alert: LokiDown
    expr: absent(up{job="kubernetes-pods", container="loki"})
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Loki недоступен"
      description: "Контейнер Loki перестал отдавать метрики. Сбор логов может быть нарушен."

  # Проверка работы агентов Promtail
  - alert: PromtailDown
    expr: absent(up{job="kubernetes-pods", container="promtail"})
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Агенты Promtail не отвечают"
      description: "Promtail перестал отправлять данные. Логи с узлов не поступают."

```

### File: ./availability.yml

```yaml
groups:
- name: availability
  rules:
  # Если сервис исчез из системы совсем
  - alert: NodeExporterMissing
    expr: absent(up{job="node-exporter"})
    for: 10s
    labels:
      severity: critical
    annotations:
      summary: "Критическая ошибка: Node Exporter исчез"
      description: "В кластере не найдено ни одного запущенного Node Exporter!"

  # Если сервис есть, но он не отвечает
  - alert: NodeExporterDown
    expr: up{job="node-exporter"} == 0
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Узел {{ $labels.instance }} недоступен"
      description: "Node Exporter на узле {{ $labels.instance }} не отвечает."

  # Новое: Проверка доступности самой Grafana
  - alert: GrafanaDown
    expr: absent(up{job="kubernetes-pods", container="grafana"})
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Панель Grafana недоступна"
      description: "Интерфейс мониторинга Grafana не отвечает."

```
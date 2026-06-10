# Развёртывание кластера на новом VPS через ArgoCD

Полное пошаговое руководство по разворачиванию production Kubernetes кластера из этого репозитория на новом сервере. Репозиторий использует паттерн **App-of-Apps** в ArgoCD — достаточно применить один root-манифест, и ArgoCD развернёт всю инфраструктуру автоматически.

---

## Архитектура репозитория

### Что входит в стек

| Компонент | Назначение | Sync Wave |
|-----------|-----------|-----------|
| **Security** | NetworkPolicy, PSA, PDB, ResourceQuota, LimitRange | -3 (первый) |
| **Sealed Secrets** | Шифрование секретов в Git | -2 |
| **Cert-Manager** | Автоматические SSL-сертификаты от Let's Encrypt | -2 |
| **MetalLB** | Bare-metal LoadBalancer (выдаёт External IP сервисам) | -1 |
| **Vault** | Хранилище секретов HashiCorp (Helm chart) | -1 |
| **Ingress-NGINX** | Входной контроллер HTTP/HTTPS трафика | -1 |
| **KEDA** | Автоскейлинг подов по метрикам (Helm chart) | 0 |
| **RBAC** | Пользователи и роли кластера | 0 |
| **Monitoring** | Prometheus + Grafana + Loki + Promtail + Alertmanager | 0 |
| **App** | Ваше приложение | 1 (последний) |

### Паттерн App-of-Apps

```
root-cluster (ArgoCD Application)
  └── смотрит в /apps/ директорию
       ├── security.yaml        → infrastructure/security/
       ├── sealed-secrets.yaml  → infrastructure/sealed-secrets/
       ├── cert-manager.yaml    → infrastructure/cert-manager/
       ├── load-balancer.yaml   → infrastructure/load-balancer/
       ├── vault.yaml           → Helm: helm.releases.hashicorp.com
       ├── ingress-nginx.yaml   → infrastructure/ingress-nginx/
       ├── keda.yaml            → Helm: kedacore.github.io/charts
       ├── rbac.yaml            → infrastructure/rbac_claster_user/
       ├── monitoring.yaml      → infrastructure/monitoring/
       └── app-deploy.yaml      → app/
```

**Sync Waves** управляют порядком деплоя: -3 → -2 → -1 → 0 → 1. Это гарантирует, что security применяется первой, сеть и сертификаты — до приложений, а приложения — в самом конце.

---

## Предварительные требования

### Серверы

- **1 мастер-нода** (минимум 2 CPU, 4 GB RAM)
- **1+ worker-нод** (минимум 2 CPU, 4 GB RAM)
- Ubuntu 24.04 LTS (или совместимый Linux)
- Публичный IP-адрес (минимум 1, лучше 2)

### Доменное имя

Нужен домен с возможностью управлять DNS-записями. Понадобятся A-записи:
- `argocd.your-domain.com` → ваш IP
- `grafana.your-domain.com` → ваш IP
- `prometheus.your-domain.com` → ваш IP
- `vault.your-domain.com` → ваш IP
- `test-app.your-domain.com` → ваш IP (или ваше имя приложения)

### Софт на серверах

- containerd (container runtime)
- kubeadm, kubelet, kubectl v1.35+
- Calico CNI

> Для автоматической установки используйте Ansible плейбук из `new_nodes/ubn24_containerd_kub1.35/`:
> ```bash
> ansible-playbook k8s-node.yml -i <worker-hostname>, -e "node_hostname=<worker-hostname> k8s_version=1.35"
> ```

---

## Этап 1: Инициализация кластера

### 1.1. Инициализация control-plane

На мастер-ноде:

```bash
kubeadm init --apiserver-advertise-address=<YOUR_MASTER_IP>
```

Скопируйте kubeconfig:

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 1.2. Подключение worker-нод

На каждой worker-ноде выполните команду `kubeadm join`, которую выдала предыдущая команда:

```bash
kubeadm join <MASTER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### 1.3. Установка Calico (CNI)

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Дождитесь готовности всех нод:

```bash
kubectl get nodes -w
# Все ноды должны быть Ready
```

### 1.4. Проверка

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
# Все системные поды должны быть Running
```

---

## Этап 2: Замена значений в репозитории

### 2.1. Форк и клонирование

```bash
# Форкните репозиторий на GitHub, затем:
git clone https://github.com/YOUR_USER/YOUR_REPO.git
cd YOUR_REPO
```

### 2.2. Автоматическая замена (скрипт)

Создайте файл `customize.sh` и выполните его:

```bash
#!/bin/bash
# === ЗАПОЛНИТЕ СВОИ ЗНАЧЕНИЯ ===
OLD_DOMAIN="example.com"
NEW_DOMAIN="your-domain.com"

OLD_REPO="https://github.com/Shum4yN1t3/k8s-gitops-platform"
NEW_REPO="https://github.com/YOUR_USER/YOUR_REPO"

OLD_MASTER_IP="203.0.113.10"
NEW_MASTER_IP="YOUR_MASTER_IP"

OLD_LB_IP="203.0.113.20"
NEW_LB_IP="YOUR_SECOND_IP"    # Второй IP для MetalLB, или тот же что master

OLD_EMAIL="admin@example.com"
NEW_EMAIL="your-email@example.com"

OLD_REGISTRY="git.example.org:5050/y.shumaylov/project-test:latest"
NEW_REGISTRY="your-registry.com/your-image:latest"

OLD_HOSTNAME="k8master"
NEW_HOSTNAME="your-master-node-name"
# === КОНЕЦ ЗНАЧЕНИЙ ===

echo "=== Замена домена ==="
find . -type f \( -name "*.yaml" -o -name "*.yml" \) -exec sed -i "s/${OLD_DOMAIN}/${NEW_DOMAIN}/g" {} +

echo "=== Замена GitHub URL ==="
find . -type f \( -name "*.yaml" -o -name "*.yml" \) -exec sed -i "s|${OLD_REPO}|${NEW_REPO}|g" {} +

echo "=== Замена IP мастера ==="
find . -type f \( -name "*.yaml" -o -name "*.yml" \) -exec sed -i "s/${OLD_MASTER_IP}/${NEW_MASTER_IP}/g" {} +

echo "=== Замена IP LoadBalancer ==="
find . -type f \( -name "*.yaml" -o -name "*.yml" \) -exec sed -i "s/${OLD_LB_IP}/${NEW_LB_IP}/g" {} +

echo "=== Замена email ==="
find . -type f \( -name "*.yaml" -o -name "*.yml" \) -exec sed -i "s/${OLD_EMAIL}/${NEW_EMAIL}/g" {} +

echo "=== Замена registry ==="
sed -i "s|${OLD_REGISTRY}|${NEW_REGISTRY}|g" app/deployment.yaml

echo "=== Замена hostname ноды ==="
find . -path "*/monitoring/*.yml" -exec sed -i "s/${OLD_HOSTNAME}/${NEW_HOSTNAME}/g" {} +

echo "=== Готово! Проверьте изменения: ==="
echo "git diff --stat"
```

```bash
chmod +x customize.sh
./customize.sh
```

### 2.3. Ручная проверка (обязательно)

После скрипта проверьте каждый файл:

```bash
# Убедитесь что старые значения не остались
grep -r "example.com" --include="*.yaml" --include="*.yml" .
grep -r "Shum4yN1t3" --include="*.yaml" --include="*.yml" .
grep -r "203.0.113.10" --include="*.yaml" --include="*.yml" .
grep -r "203.0.113.20" --include="*.yaml" --include="*.yml" .
```

Все команды должны вернуть пустой результат.

### 2.4. Полная карта замен (для ручной замены)

#### IP-адреса

| Файл | Что заменить | На что |
|------|-------------|--------|
| `infrastructure/load-balancer/metallb-pool.yaml` | `203.0.113.10/32` | `<YOUR_MASTER_IP>/32` |
| `infrastructure/load-balancer/metallb-pool.yaml` | `203.0.113.20/32` | `<YOUR_SECOND_IP>/32` |
| `infrastructure/security/network-policies/ingress-nginx-netpol.yaml` | `203.0.113.10/32` | `<YOUR_MASTER_IP>/32` |

> **MetalLB IP Pool:** Если у вас только один IP, укажите его дважды. Если есть диапазон (например, 10 IP), используйте формат `START_IP-END_IP`.

#### Домен (5 поддоменов)

| Файл | Текущий хост | Заменить на |
|------|-------------|-------------|
| `app/ingress-app.yaml` | `test-app.example.com` | `app.your-domain.com` |
| `argocd/argo_ingress.yml` | `argocd.example.com` | `argocd.your-domain.com` |
| `infrastructure/monitoring/05-ingress-grafana.yml` | `grafana.example.com` | `grafana.your-domain.com` |
| `infrastructure/monitoring/05-ingress-grafana.yml` | `prometheus.example.com` | `prometheus.your-domain.com` |
| `infrastructure/vault/vault-app.yml` + `apps/vault.yaml` | `vault.example.com` | `vault.your-domain.com` |

#### Email

| Файл | Текущее | Заменить на |
|------|---------|-------------|
| `infrastructure/cert-manager/cluster_issuer.yml` | `admin@example.com` | Ваш email |

#### GitHub репозиторий

| Файл | Заменить `repoURL` |
|------|-------------------|
| `argocd/root-app.yml` | `https://github.com/YOUR_USER/YOUR_REPO.git` |
| `apps/app-deploy.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |
| `apps/cert-manager.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |
| `apps/ingress-nginx.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |
| `apps/keda.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |
| `apps/load-balancer.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |
| `apps/monitoring.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |
| `apps/rbac.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |
| `apps/sealed-secrets.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |
| `apps/security.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |
| `apps/vault.yaml` | `https://github.com/YOUR_USER/YOUR_REPO` |

#### Container Registry

| Файл | Текущее | Заменить на |
|------|---------|-------------|
| `app/deployment.yaml` | `git.example.org:5050/y.shumaylov/project-test:latest` | `your-registry/your-image:tag` |

#### Hostname ноды

| Файл | Текущее | Заменить на |
|------|---------|-------------|
| `infrastructure/monitoring/03-main-stack.yml` | `kubernetes.io/hostname: k8master` | hostname вашей ноды |
| `infrastructure/monitoring/08-loki.yml` | `kubernetes.io/hostname: k8master` | hostname вашей ноды |

Узнать hostname:
```bash
kubectl get nodes -o wide
# Столбец NAME — это hostname
```

### 2.5. Закоммитить и запушить

```bash
git add -A
git commit -m "Customize: replace cluster-specific values for new VPS"
git push origin main
```

---

## Этап 3: Установка ArgoCD

### 3.1. Создание namespace и установка

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3.2. Дождаться готовности

```bash
kubectl get pods -n argocd -w
# Все поды должны стать Running (3-5 минут)
```

### 3.3. Получить пароль администратора

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 3.4. Настроить доступ (временный port-forward)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Открыть https://localhost:8080
# Логин: admin / пароль из предыдущего шага
```

---

## Этап 4: Запуск инфраструктуры (один манифест)

### 4.1. Применить root application

```bash
kubectl apply -f argocd/root-app.yml
```

Это запустит цепную реакцию:
1. ArgoCD создаст root-cluster Application
2. root-cluster просканирует директорию `/apps/`
3. Найдёт 10 YAML файлов → создаст 10 child Applications
4. Каждая Application начнёт синхронизировать свой путь из Git
5. Sync Waves обеспечат правильный порядок

### 4.2. Наблюдение за деплоем

```bash
# Статус всех приложений
kubectl get applications -n argocd -w

# Ожидаемый результат через 5-10 минут:
# NAME            SYNC STATUS   HEALTH STATUS
# app-deploy      Synced        Healthy
# certmanager     Synced        Healthy
# ingressnginx    Synced        Healthy
# keda            Synced        Healthy
# loadbalancer    Synced        Healthy
# monitoring      Synced        Healthy
# rbacuser        Synced        Healthy
# root            Synced        Healthy
# sealedsecrets   Synced        Healthy
# security        Synced        Healthy
# vault           Synced        Healthy
```

### 4.3. Применить Ingress для ArgoCD

```bash
kubectl apply -f argocd/argo_ingress.yml
```

> **Почему отдельно?** Ingress для ArgoCD зависит от cert-manager и ingress-nginx, которые ещё деплоятся. Применяйте после того, как certmanager и ingressnginx станут Healthy.

---

## Этап 5: Пересоздание секретов

### 5.1. Docker Registry Secret

```bash
kubectl create secret docker-registry regcred \
  --docker-server=YOUR_REGISTRY_URL \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_TOKEN \
  --docker-email=YOUR_EMAIL \
  -n app
```

### 5.2. Sealed Secrets

Sealed Secrets зашифрованы ключом конкретного кластера. На новом кластере нужно пересоздать их.

**Grafana admin пароль:**
```bash
# Создать обычный Secret
kubectl create secret generic grafana-admin-secret \
  --from-literal=admin-password=YOUR_GRAFANA_PASSWORD \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace kube-system -o yaml \
  > infrastructure/monitoring/grafana-sealed-secret.yaml

# Закоммитить
git add infrastructure/monitoring/grafana-sealed-secret.yaml
git commit -m "Re-seal grafana secret for new cluster"
git push
```

> **Примечание:** `kubeseal` требует работающий Sealed Secrets controller в кластере. Он уже будет установлен через ArgoCD к этому моменту.

---

## Этап 6: Настройка DNS

Создайте A-записи у вашего DNS-провайдера:

| Запись | Тип | Значение |
|--------|-----|---------|
| `argocd.your-domain.com` | A | `<YOUR_MASTER_IP>` |
| `grafana.your-domain.com` | A | `<YOUR_MASTER_IP>` |
| `prometheus.your-domain.com` | A | `<YOUR_MASTER_IP>` |
| `vault.your-domain.com` | A | `<YOUR_MASTER_IP>` |
| `app.your-domain.com` | A | `<YOUR_MASTER_IP>` |

> Если MetalLB выдаёт другой External IP для ingress-nginx, используйте его:
> ```bash
> kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
> ```

DNS-записи начнут работать через несколько минут. После этого cert-manager автоматически выпустит SSL-сертификаты от Let's Encrypt.

---

## Этап 7: Проверка

### 7.1. Все компоненты работают

```bash
# Ноды
kubectl get nodes -o wide

# ArgoCD приложения
kubectl get applications -n argocd

# Все поды
kubectl get pods -A | grep -v Completed

# Ingress-nginx
kubectl get pods -n ingress-nginx
# Должен быть 1/1 Running

# Мониторинг
kubectl get pods -n monitoring
# prometheus, grafana, loki, promtail, alertmanager, node-exporter, kube-state-metrics

# Сертификаты
kubectl get certificates -A
# Все должны быть Ready=True
```

### 7.2. Доступность через браузер

- **ArgoCD:** `https://argocd.your-domain.com`
- **Grafana:** `https://grafana.your-domain.com`
- **Prometheus:** `https://prometheus.your-domain.com`
- **Vault:** `https://vault.your-domain.com`
- **Приложение:** `https://app.your-domain.com`

### 7.3. Проверка NetworkPolicy

```bash
kubectl get networkpolicy -A
# Должны быть NetworkPolicy для: ingress-nginx, monitoring, cert-manager, metallb, vault, app
```

---

## Типичные проблемы при развёртывании

### Cert-manager не выдаёт сертификаты

```bash
kubectl get challenges -A
kubectl describe challenge -n <namespace> <challenge-name>
```

Частые причины:
- DNS ещё не прописан или не обновился
- Порт 80 заблокирован файрволом (нужен для HTTP-01 challenge)
- Email не указан в ClusterIssuer

### Ingress-nginx в CrashLoopBackOff

Скорее всего, NetworkPolicy блокирует egress к API серверу. Проверьте:
```bash
kubectl logs -n ingress-nginx <pod-name>
# Если обрывается на "Creating API client" — проблема в NetworkPolicy
```

Решение описано в файле `Troubleshooting-and-Customization.md`.

### MetalLB не назначает External IP

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
# Если EXTERNAL-IP в состоянии <pending>:
kubectl get ipaddresspool -n metallb-system
kubectl logs -n metallb-system -l app=metallb,component=controller
```

Проверьте что IP в `metallb-pool.yaml` реально принадлежит вашему серверу.

### Vault не инициализирован

Vault после установки требует инициализации:
```bash
kubectl exec -n vault vault-0 -- vault operator init
# Сохраните Unseal Keys и Root Token!

kubectl exec -n vault vault-0 -- vault operator unseal <key1>
kubectl exec -n vault vault-0 -- vault operator unseal <key2>
kubectl exec -n vault vault-0 -- vault operator unseal <key3>
```

### ArgoCD Application в состоянии Progressing

```bash
kubectl get applications -n argocd
# Если какое-то приложение Progressing долго:
kubectl describe application <app-name> -n argocd
# Посмотреть Last Sync Status и Conditions
```

Принудительный sync:
```bash
kubectl annotate application <app-name> -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

---

## Что НЕ нужно менять

Эти значения стандартны для Kubernetes и одинаковы на любом кластере:

| Значение | Что это |
|---------|---------|
| `10.96.0.1` | ClusterIP сервиса kubernetes (стандарт kubeadm) |
| `10.96.0.10` | ClusterIP сервиса kube-dns |
| `*.svc.cluster.local` | Внутренний DNS-суффикс Kubernetes |
| Helm chart URLs | kedacore.github.io, helm.releases.hashicorp.com |
| Container images (nginx, prometheus, grafana...) | Публичные образы из registry.k8s.io, prom/, grafana/ |

> **Исключение:** Если при `kubeadm init` вы указали `--service-cidr` отличный от `10.96.0.0/12`, то адреса `10.96.0.1` и `10.96.0.10` будут другими. В этом случае обновите их в NetworkPolicy.

---

## Чеклист

```
[ ]  1. Серверы подготовлены (containerd, kubeadm, kubelet)
[ ]  2. kubeadm init выполнен, worker-ноды подключены
[ ]  3. Calico CNI установлен, все ноды Ready
[ ]  4. Репозиторий форкнут и значения заменены
[ ]  5. DNS A-записи созданы
[ ]  6. ArgoCD установлен
[ ]  7. root-app.yml применён
[ ]  8. Все ArgoCD Applications в статусе Synced/Healthy
[ ]  9. Docker Registry Secret создан
[ ] 10. Sealed Secrets пересозданы
[ ] 11. ArgoCD Ingress применён
[ ] 12. SSL-сертификаты выпущены (kubectl get certificates -A)
[ ] 13. Vault инициализирован и распечатан (unseal)
[ ] 14. Все сервисы доступны через браузер
```

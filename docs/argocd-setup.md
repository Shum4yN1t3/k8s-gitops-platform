# Описание и порядок настройки ArgoCD

ArgoCD используется как основной инструмент GitOps. Он следит за состоянием репозитория в GitHub и автоматически синхронизирует изменения с вашим кластером. Развертывание выполняется в четыре основных этапа.

## Порядок запуска и настройки

### 1. Подготовка и базовая установка

Сначала создается выделенное пространство имен и устанавливаются стандартные компоненты ArgoCD.

```bash
# Создание пространства имен
kubectl create namespace argocd

# Установка компонентов ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```

### 2. Внешний доступ и SSL (argo_ingress.yml)

**Что делает:** Создает точку входа в веб-интерфейс ArgoCD.

* **SSL Passthrough:** Важная настройка, так как сервер ArgoCD по умолчанию сам обрабатывает TLS.
* **Backend Protocol HTTPS:** Указывает контроллеру NGINX, что общаться с сервисом нужно по защищенному протоколу.
* **Cert-manager:** Автоматически выпускает сертификат для домена `argocd.example.com`.

**Содержимое файла:**

```yaml
### File: ./argo_ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-server-tls

```

### 3. Паттерн «App of Apps» (root-app.yml)

**Что делает:** Это "корневое" приложение. Оно указывает ArgoCD на ваш репозиторий `k8s-gitops-platform.git` и заставляет его рекурсивно искать другие манифесты внутри папок.

* **ignoreDifferences:** Настроено для MetalLB. Это исключает конфликты синхронизации, когда кубернетис сам генерирует CA-бандлы для вебхуков, а ArgoCD пытается их "откатить" к состоянию из Git.
* **Automated Sync:** Включено автоматическое исправление (selfHeal) и удаление ресурсов, которых больше нет в Git (prune).

**Содержимое файла:**

```yaml
### File: ./root-app.yml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-cluster
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Shum4yN1t3/k8s-gitops-platform.git
    targetRevision: HEAD
    path: .
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  ignoreDifferences:
    - group: admissionregistration.k8s.io
      kind: ValidatingWebhookConfiguration
      name: metallb-webhook-configuration
      jsonPointers:
        - /webhooks/0/clientConfig/caBundle
        - /webhooks/1/clientConfig/caBundle
    - group: apiextensions.k8s.io
      kind: CustomResourceDefinition
      jsonPointers:
        - /spec/conversion/webhook/clientConfig/caBundle

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```

---

## 🔑 Первичный вход в систему

После применения всех манифестов, ArgoCD создаёт временный пароль для администратора. Чтобы войти в панель через браузер, выполните команду:

```bash
# Получение пароля администратора
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Логин:** `admin`
**URL:** `https://argocd.example.com`

## 🛠️ Команды применения

После подготовки файлов в папке `~/k8s_yml/argocd/`, деплой выполняется так:

```bash
kubectl apply -f argo_ingress.yml
kubectl apply -f root-app.yml

```

Теперь любое изменение в репозитории `k8s-gitops-platform.git` будет автоматически применяться в кластере.
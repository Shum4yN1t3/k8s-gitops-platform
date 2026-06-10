# Миграция CI/CD с GitLab на GitHub — полный переход на GHCR

**Дата:** 25.03.2026  
**Кластер:** k8master + k8worker2  
**Контекст:** Завершение миграции с GitLab CI/CD + GitLab Container Registry на GitHub Actions + GHCR. Устранение каскада проблем при деплое приложения project-test.

---

## 1. Предыстория — что было до миграции

### Старая схема (GitLab)

```
GitLab CI → build image → push to git.example.org:5050 → regcred (GitLab PAT) → k8s pull image
```

- Образы хранились в GitLab Container Registry (`git.example.org:5050`)
- `regcred` секрет был создан для GitLab:
  ```bash
  kubectl create secret docker-registry regcred \
    --docker-server=git.example.org:5050 \
    --docker-username=k8s-image-pull \
    --docker-password=<GITLAB_PAT> \
    --docker-email=none@example.com \
    -n default
  ```
- SealedSecret лежал в `infrastructure/` с `namespace: default`

### Новая схема (GitHub)

```
GitHub Actions → build image → push to ghcr.io → regcred (GitHub PAT) → k8s pull image
```

- Образы в GitHub Container Registry (`ghcr.io/shum4yn1t3/project-test`)
- CI/CD через GitHub Actions в репо `project-iac`
- Обновление тега образа через `repository_dispatch` в `k8s-gitops-platform`

---

## 2. Проблема — AppNoPods: ни одного пода в namespace app

### Алерт

```
🔥 АЛАРМ: critical
Событие: AppNoPods
Описание: В неймспейсе app не найдено ни одного пода
Статус: FIRING
Начало: 17:37:16 24.03 UTC
```

### Диагностика

При попытке создания подов ReplicaSet controller выдавал:

```
Warning  FailedCreate  replicaset-controller
Error creating: pods "project-test-78dc5cc447-5dgdn" is forbidden:
violates PodSecurity "restricted:latest":
  - allowPrivilegeEscalation != false
  - unrestricted capabilities
  - runAsNonRoot != true
  - seccompProfile not set
```

Неймспейс `app` имеет PodSecurity policy `restricted:latest`, а deployment не имел `securityContext`.

### Решение — securityContext

Добавили в `app/deployment.yaml`:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: project-test
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
```

**Важно:** `runAsUser: 65532` обязателен — образ использует non-numeric user `nonroot`, и Kubernetes не может проверить `runAsNonRoot: true` без явного UID. Без этого поля — `CreateContainerConfigError`.

---

## 3. Проблема — regcred SealedSecret: неправильный registry и namespace

### Что было не так

1. **Неправильный registry** — старый `regcred` был для `git.example.org:5050` (GitLab), а образы теперь на `ghcr.io`
2. **Неправильный namespace** — SealedSecret имел `namespace: default`, а поды в `namespace: app`
3. **Не деплоился** — SealedSecret лежал в `infrastructure/`, но не был включён ни в один ArgoCD Application → секрет никогда не применялся
4. **Broken scope** — SealedSecret был создан со strict scope (default), пересоздание с другим namespace ломает расшифровку → `ErrUnsealFailed: no key could decrypt secret`

### Решение — обычный секрет через kubectl

Отказались от SealedSecret для regcred — создали секрет напрямую:

```bash
# Удаляем старый sealed secret и secret
kubectl delete sealedsecret regcred -n app --ignore-not-found
kubectl delete secret regcred -n app --ignore-not-found

# Создаём новый секрет для ghcr.io
kubectl create secret docker-registry regcred \
  --docker-server=ghcr.io \
  --docker-username=Shum4yN1t3 \
  --docker-password=<GITHUB_PAT> \
  --namespace=app

# Защищаем от удаления при ArgoCD sync
kubectl annotate secret regcred -n app \
  argocd.argoproj.io/compare-options=IgnoreExtraneous
```

Удалили `app/regcred-sealed.yaml` из репо чтобы ArgoCD не пытался применить сломанный SealedSecret.

---

## 4. Проблема — CreateContainerConfigError после runAsNonRoot

### Ошибка

```
Warning  Failed  kubelet
Error: container has runAsNonRoot and image has non-numeric user (nonroot),
cannot verify user is non-root
```

### Причина

Образ `ghcr.io/shum4yn1t3/project-test` собран с `USER nonroot` (строка, не число). Kubernetes требует numeric UID для проверки `runAsNonRoot`.

### Решение

Добавили `runAsUser: 65532` в pod securityContext. `65532` — стандартный UID для пользователя `nonroot` в distroless/chainguard образах.

---

## 5. Проблема — ErrUnsealFailed

### Ошибка в ArgoCD

```
regcred  degraded
ErrUnsealFailed: Failed to unseal: no key could decrypt secret (.dockerconfigjson)
```

### Причина

SealedSecret был запечатан для другого namespace (strict scope по умолчанию привязывает encrypted data к конкретному name + namespace). Смена namespace с `default` на `app` сломала расшифровку.

### Решение

Удалили `app/regcred-sealed.yaml` из репо. Секрет создан напрямую через `kubectl` (см. пункт 3).

---

## 6. Итог миграции

### Что изменилось

| Компонент | Было (GitLab) | Стало (GitHub) |
|---|---|---|
| CI/CD | GitLab CI | GitHub Actions |
| Container Registry | `git.example.org:5050` | `ghcr.io` |
| Image pull secret | SealedSecret → GitLab PAT | kubectl secret → GitHub PAT |
| Secret namespace | `default` (не деплоился) | `app` (напрямую в кластере) |
| Image update | Ручной / GitLab CI | `repository_dispatch` → `update-image.yml` |

### Файлы изменённые в k8s-gitops-platform

- `app/deployment.yaml` — добавлен securityContext (PodSecurity restricted)
- `app/regcred-sealed.yaml` — удалён (заменён на kubectl secret)
- `.github/workflows/update-image.yml` — workflow для обновления image tag по событию `repository_dispatch`

### Текущее состояние

- Поды `project-test` в namespace `app` — **Running**
- ArgoCD `app-deploy` — **Synced / Healthy**
- Образы тянутся с `ghcr.io` через `regcred` секрет с GitHub PAT

---

## 7. Выводы и правила на будущее

### PodSecurity restricted — чеклист для deployment

Любой deployment в namespace с `restricted` policy должен иметь:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532        # обязательно для non-numeric USER в образе
        seccompProfile:
          type: RuntimeDefault
      containers:
        - securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
```

### SealedSecrets — когда НЕ использовать

- Если секрет нужен в namespace отличном от того, для которого он запечатан — нужно **пересоздать** через `kubeseal --namespace <new-ns>`
- Если нет доступа к `kubeseal` или sealed-secrets controller ещё не настроен — создать секрет через `kubectl` + аннотация `argocd.argoproj.io/compare-options=IgnoreExtraneous`
- Для image pull secrets проще использовать `kubectl create secret` напрямую

### Миграция container registry — чеклист

1. Пересобрать и запушить образы в новый registry
2. Обновить `image:` в deployment манифестах
3. Пересоздать `regcred` секрет для нового registry
4. Проверить что поды могут pull-ить образ
5. Ротировать старые токены/PAT

### Ротация PAT

После завершения миграции рекомендуется ротировать GitHub PAT который использовался для создания `regcred`, если он был засвечен в логах или чатах.

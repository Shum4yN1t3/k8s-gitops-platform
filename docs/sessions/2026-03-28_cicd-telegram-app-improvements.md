# CI/CD Telegram-нотификации и улучшения Go-приложения

**Дата:** 28.03.2026  
**Кластер:** k8master + k8worker2  
**Контекст:** Добавление Telegram-уведомлений в CI/CD пайплайны, расширение функциональности Go-приложения project-test, настройка интервала синхронизации ArgoCD.

---

## 1. Telegram-уведомления в CI/CD

### Проблема

При деплое приложения не было никаких уведомлений — чтобы узнать статус, приходилось заходить в GitHub Actions UI вручную.

### Решение

Добавили Telegram-нотификации в оба пайплайна. Используется тот же бот что и в AlertManager (`GitLabLivenessBot`).

#### Секреты (добавлены в оба репо)

- `TELEGRAM_BOT_TOKEN` — токен Telegram-бота
- `TELEGRAM_CHAT_ID` — ID чата для уведомлений

Добавлены вручную: GitHub → Settings → Secrets → Actions (в обоих репозиториях).

#### project-iac: `.github/workflows/ci.yml`

Три новых job'а:

1. **`notify-start`** — при push в main сразу отправляет:
   ```
   building sha-XXXXXXX...
   Shum4yN1t3: commit message
   ```

2. **`notify-success`** — после успешного деплоя с длительностью:
   ```
   sha-XXXXXXX deployed (1m 23s)
   ```

3. **`notify-failure`** — при ошибке на любом шаге:
   ```
   FAIL sha-XXXXXXX
   Shum4yN1t3: commit message
   logs
   ```

#### k8s-gitops-platform: `.github/workflows/update-image.yml`

Два новых step'а в конце job'а `update-deployment`:

- **Success:** `k8s updated to sha-XXXXXXX`
- **Failure:** `k8s update FAIL sha-XXXXXXX` + ссылка на логи

### Полный flow уведомлений

```
push в project-iac/main
  → Telegram: "building sha-XXX..."
  → тесты + билд + push образа
  → Telegram: "sha-XXX deployed (Xm Xs)"
  → repository_dispatch → k8s-gitops-platform
  → Telegram: "k8s updated to sha-XXX"
  → ArgoCD sync → поды обновлены
```

Используется GitHub Action: `appleboy/telegram-action@master`.

---

## 2. Улучшения Go-приложения (project-test)

### 2.1. Structured request logging

**Файл:** `main.go`

Каждый HTTP-запрос логируется в JSON через Zap:

```json
{"level":"info","msg":"http_request","request_id":"a1b2c3d4e5f6g7h8","method":"GET","path":"/api/ping","status":200,"duration_seconds":0.0001}
```

- `request_id` генерируется через `crypto/rand` (8 байт → hex)
- Header `X-Request-ID` добавляется в каждый ответ для трейсинга
- `/healthz`, `/readyz`, `/metrics` не логируются (шум от проб и скрейпера)
- Логи идут в stdout → Promtail → Loki → Grafana

### 2.2. pprof profiling endpoints

**Файл:** `main.go`

Подключён `net/http/pprof` из стандартной библиотеки:

- `/debug/pprof/` — индекс
- `/debug/pprof/profile` — CPU profile
- `/debug/pprof/heap` — heap dump
- `/debug/pprof/goroutine` — goroutine dump
- `/debug/pprof/trace` — execution trace

Использование:
```bash
go tool pprof https://test-app.example.com/debug/pprof/heap
```

### 2.3. Rate limiter middleware

**Файл:** `ratelimit.go` (новый)

Token bucket rate limiter per IP:

- Конфигурация через ENV: `RATE_LIMIT` (req/sec, default 100), `RATE_BURST` (burst, default 200)
- При превышении лимита → `429 Too Many Requests` + header `Retry-After: 1`
- Prometheus метрика `http_rate_limited_total` для мониторинга
- Background goroutine чистит неактивные IP (>10 мин) каждую минуту
- Защищает: `/`, `/api/ping`, `/web`, `/web/events`
- Не защищает: `/healthz`, `/readyz`, `/metrics`, `/debug/pprof/*`

### 2.4. Live SSE dashboard

**Файл:** `web.go` (переписан)

Заменён meta-refresh (перезагрузка каждые 5с) на Server-Sent Events:

- **`/web`** — HTML-страница с JavaScript + Chart.js (CDN)
- **`/web/events`** — SSE endpoint, стримит JSON каждую 1 секунду
- Три live-графика (последние 60 точек):
  - Memory (Heap In-Use)
  - Goroutines
  - Requests Per Second
- Числовые значения обновляются мгновенно без перезагрузки
- Индикатор подключения (зелёная пульсирующая точка)
- Тёмная тема в стиле GitHub
- При потере соединения — автоматический реконнект (встроено в SSE)

### Новые файлы в project-iac

| Файл | Описание |
|------|----------|
| `ratelimit.go` | Token bucket rate limiter per IP |
| `ratelimit_test.go` | Тесты: burst, refill, разные IP, concurrency |

### Изменённые файлы в project-iac

| Файл | Что изменилось |
|------|----------------|
| `main.go` | request logging, pprof, rate limiter, config (RateLimit/RateBurst) |
| `web.go` | Переписан: SSE + Chart.js вместо meta-refresh |
| `web_test.go` | Тесты SSE endpoint, Chart.js, pprof ссылка |
| `main_test.go` | Тесты X-Request-ID, pprof, rate limiter 429 |
| `.github/workflows/ci.yml` | Telegram-нотификации |

---

## 3. ArgoCD — уменьшение интервала синхронизации

### Проблема

После push в k8s-gitops-platform ArgoCD синхронизировался только через 3 минуты (дефолтный `timeout.reconciliation: 180s`).

### Решение

```bash
kubectl patch configmap argocd-cm -n argocd \
  --type merge -p '{"data":{"timeout.reconciliation":"60s"}}'
```

Теперь ArgoCD проверяет git каждые 60 секунд. Изменение подхватывается без рестарта.

**Важно:** `argocd-cm` — системный ConfigMap ArgoCD, не управляется через git. Меняется напрямую в кластере.

---

## 4. Исправление CI: lowercase image tags

### Проблема

```
ERROR: failed to build: invalid tag "ghcr.io/Shum4yN1t3/project-test:sha-16f9172":
repository name must be lowercase
```

`github.repository_owner` возвращает `Shum4yN1t3` с заглавными буквами, а Docker теги требуют lowercase.

### Решение

В step `Set variables` добавлен `tr '[:upper:]' '[:lower:]'`:

```yaml
- name: Set variables
  id: vars
  run: |
    echo "short_sha=sha-$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT
    echo "image_name=$(echo ${{ env.IMAGE_NAME }} | tr '[:upper:]' '[:lower:]')" >> $GITHUB_OUTPUT
```

---

## 5. Итоговое состояние

### Endpoints приложения project-test

| Endpoint | Описание |
|----------|----------|
| `/` | JSON pong |
| `/api/ping` | JSON pong |
| `/healthz` | Liveness probe |
| `/readyz` | Readiness probe |
| `/metrics` | Prometheus метрики |
| `/web` | Live SSE dashboard с графиками |
| `/web/events` | SSE stream метрик (1/sec) |
| `/debug/pprof/` | Go profiling |

### CI/CD pipeline

```
git push (project-iac/main)
  → Telegram: "building sha-XXX..."
  → go test -v -race
  → docker build + push → ghcr.io
  → repository_dispatch → k8s-gitops-platform
  → Telegram: "sha-XXX deployed (Xm Xs)"
  → sed update deployment.yaml + git push
  → Telegram: "k8s updated to sha-XXX"
  → ArgoCD sync (≤60s)
  → новые поды запущены
```

### Prometheus метрики приложения

- `http_requests_total` — счётчик запросов (method, endpoint, status)
- `http_request_duration_seconds` — гистограмма латенции
- `http_5xx_errors_total` — счётчик 5xx ошибок
- `http_rate_limited_total` — счётчик заблокированных запросов

---

## 6. Выводы и правила на будущее

### Docker image tags

Всегда приводить `IMAGE_NAME` к lowercase через `tr` — `github.repository_owner` может содержать заглавные буквы.

### Telegram-нотификации в CI/CD

Минимум информации, максимум пользы:
- Начало: что билдится и кем
- Конец: тег + длительность
- Ошибка: тег + ссылка на логи

### ArgoCD reconciliation

- Дефолт 180s слишком долго для активной разработки
- 60s — хороший баланс между скоростью и нагрузкой на git
- Для мгновенной синхронизации — настроить GitHub webhook → ArgoCD

### Rate limiting

- Конфигурировать через ENV (не хардкодить)
- Не ограничивать health-check и metrics эндпоинты
- Мониторить через Prometheus метрику `http_rate_limited_total`

### SSE vs polling

- SSE лучше meta-refresh: меньше трафика, мгновенные обновления, нет мигания страницы
- Использовать `context.Done()` для cleanup при disconnect
- Клиент автоматически реконнектится (встроено в EventSource API)

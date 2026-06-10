# k8s-gitops-platform

Это публичная санитизированная копия рабочего репозитория — боевой
кластер сейчас синхронизируется с приватным оригиналом. История
пересобрана с нуля: без учётных данных, реальных IP и доменов.

GitOps-платформа на «голом» Kubernetes (kubeadm, 2 ноды на VPS): ArgoCD
app-of-apps, полный мониторинг-стек, sealed-secrets, network policies,
canary-деплой через Argo Rollouts с автоматическим rollback по SLO-метрикам.

Репозиторий — единственный источник истины для кластера: ArgoCD
синхронизирует всё содержимое автоматически (`prune` + `selfHeal`).

Все публичные IP, домены и контактные данные в манифестах и документации
заменены на placeholder'ы (`203.0.113.0/24`, `example.com`).

## Стек и версии

| Компонент | Версия | Источник |
|---|---|---|
| Kubernetes | v1.35 (kubeadm) | docs/sessions/2026-03-11 |
| CNI | Calico <!-- TODO: версия не зафиксирована — манифест ставился без пина (docs/archive/2026-03-05) --> | docs/archive/2026-03-05 |
| ArgoCD | stable manifests <!-- TODO: версия не зафиксирована — установка из argo-cd/stable/manifests/install.yaml --> | docs/argocd-setup.md |
| Argo Rollouts | v1.9.0 (helm chart 2.41.0) | apps/argo-rollouts.yaml |
| ingress-nginx | v1.8.1 | infrastructure/ingress-nginx |
| MetalLB (L2) | v0.13.12 | infrastructure/load-balancer |
| cert-manager | v1.13.3 | infrastructure/cert-manager |
| sealed-secrets | v0.24.5 | infrastructure/sealed-secrets |
| KEDA | helm chart 2.16.1 | apps/keda.yaml |
| Vault | helm chart 0.28.0 | apps/vault.yaml |
| Vaultwarden | 1.32.7 | apps/warden |
| Prometheus / Grafana / Alertmanager / node-exporter | `latest` (не запинено — осознанный долг) | infrastructure/monitoring |
| kube-state-metrics | v2.13.0 | infrastructure/monitoring |
| Loki / Promtail | 2.9.2 | infrastructure/monitoring |
| blackbox-exporter / json-exporter | v0.25.0 / v0.6.0 | infrastructure/monitoring |

## Структура репозитория

```
.
├── argocd/                  # root Application (app-of-apps) + Ingress ArgoCD
├── apps/                    # ArgoCD Applications: платформа и приложение
│   └── warden/              # манифесты Vaultwarden (kustomize)
├── app/                     # workload project-test
│   ├── rollout.yaml         # Argo Rollout (canary 20% -> 50% -> 100%)
│   ├── analysis-template.yaml  # SLO-анализ canary (Prometheus)
│   ├── keda-config.yaml     # автоскейлинг (KEDA -> Rollout)
│   ├── service.yaml
│   └── ingress-app.yaml
├── infrastructure/          # платформенные компоненты (kustomize)
│   ├── cert-manager/        # + ClusterIssuer Let's Encrypt
│   ├── ingress-nginx/
│   ├── load-balancer/       # MetalLB L2
│   ├── monitoring/          # Prometheus, Grafana, Alertmanager, Loki, exporters
│   ├── sealed-secrets/
│   ├── security/            # NetworkPolicies, PDB, PSA, LimitRange/Quota
│   ├── vault/               # ArgoCD Application HashiCorp Vault
│   └── rbac_claster_user/   # ограниченный read-only доступ для bastion
└── docs/                    # incidents (RCA), sessions, archive
```

## GitOps flow

1. `argocd/root-app.yml` — root Application, указывает на каталог `apps/`.
2. Каждый файл в `apps/` — ArgoCD Application (платформенный компонент или
   приложение) с sync-wave для порядка развёртывания: security (-3) →
   sealed-secrets (-2) → argo-rollouts (-1) → инфраструктура (0) → app (1).
3. CI приложения (`.github/workflows/update-image.yml`) по
   `repository_dispatch` обновляет тег образа в `app/rollout.yaml` — дальше
   выкатку ведёт ArgoCD + Argo Rollouts.

## Progressive delivery (Argo Rollouts)

- Контроллер: `apps/argo-rollouts.yaml` (helm chart 2.41.0 = v1.9.0).
- Workload: `app/rollout.yaml` — canary-стратегия
  `20% → анализ → 50% → анализ → 100%` (без trafficRouting, доля канарейки
  управляется репликами).
- Анализ: `app/analysis-template.yaml` — три метрики из Prometheus
  (доля 5xx < 5%, p95 < 500ms, p99 < 1s за 5-минутное окно),
  `failureLimit: 1` → автоматический rollback при деградации.

## Мониторинг и SLO

- Конфигурация Prometheus: `infrastructure/monitoring/02-config.yml`
  (service discovery нод/подов, ArgoCD, blackbox/json-exporter probes).
- Правила алертинга: `infrastructure/monitoring/07-prom-rules.yml` —
  ноды, workloads, ресурсы контейнеров, self-monitoring, ArgoCD и
  SLO приложения (группа `app_slo`):
  - `app:http_requests:rate5m` — RPS;
  - `app:http_requests_5xx_ratio:rate5m` — доля 5xx за 5m;
  - `app:http_request_duration_seconds:p95_5m` / `:p99_5m` —
    `histogram_quantile` по `http_request_duration_seconds_bucket`.
- Дашборд SLO: `infrastructure/monitoring/13-grafana-dashboards.yml`
  (file-provisioning Grafana, те же recording-правила).
- Алерты доставляются в Telegram через Alertmanager; конфиг с токеном
  хранится как SealedSecret (`alertmanager-sealed-secret.yaml`,
  шаблон — `alertmanager.example.yml`).

## Безопасность

- Секреты только в виде SealedSecret (Grafana admin, Alertmanager Telegram,
  regcred); сырые ключи/kubeconfig в репозиторий не попадают (`.gitignore`).
- Default-deny NetworkPolicies по namespace'ам с точечными разрешениями
  (`infrastructure/security/network-policies/`).
- Pod Security Admission, PDB, LimitRange/ResourceQuota
  (`infrastructure/security/`).
- Workload запускается non-root, без capabilities, с seccomp RuntimeDefault.
- CI: Trivy (config + fs) и Checkov на каждый push/PR.

## Инциденты (RCA)

| Дата | Инцидент | Root cause (одной строкой) |
|---|---|---|
| 2026-03-17 | [kube-apiserver не поднялся после рестарта мастера](docs/incidents/2026-03-17_master-restart-cert-recovery.md) | После рестарта в `/etc/kubernetes/pki` отсутствовали `apiserver.*` и `front-proxy-client.*`; восстановлено `kubeadm init phase certs all` |
| 2026-03-17 | [ingress-nginx CrashLoopBackOff](docs/incidents/2026-03-17_ingress-nginx-netpol-crash.md) | Calico применяет NetworkPolicy после DNAT: egress был разрешён на ClusterIP API-сервера, а реальный пакет шёл на IP ноды:6443 и блокировался |
| 2026-03-18 | [cert-manager-cainjector CrashLoopBackOff](docs/incidents/2026-03-18_cert-manager-cainjector-netpol-crash.md) | Тот же пост-DNAT эффект Calico — в egress не был разрешён реальный IP мастера:6443 |
| 2026-04-13 | [ACME HTTP-01 challenge вечно pending](docs/incidents/2026-04-13_acme-hairpin-coredns-fix.md) | Hairpin NAT MetalLB L2: под не достучивается до собственного node-IP; решение — split-horizon DNS в CoreDNS (hosts-блок → ClusterIP ingress-controller) |

## Рабочие сессии

- [2026-03-11 — KEDA, cAdvisor, kube-state-metrics](docs/sessions/2026-03-11_keda-cadvisor-ksm-setup.md)
- [2026-03-18 — Обслуживание кластера](docs/sessions/2026-03-18_cluster-maintenance.md)
- [2026-03-25 — Миграция GitLab → GitHub](docs/sessions/2026-03-25_migration-from-gitlab-to-github.md)
- [2026-03-28 — CI/CD, Telegram-нотификации, метрики приложения](docs/sessions/2026-03-28_cicd-telegram-app-improvements.md)
- [2026-04-09 — Vaultwarden, мониторинг, ArgoCD notifications](docs/sessions/2026-04-09_vaultwarden-monitoring-notifications.md)
- [2026-04-12 — Webhook x509 / cert-manager](docs/sessions/2026-04-12_webhook-x509-certmanager-fix.md)
- [2026-04-13 — cert-manager NetPol + ClusterIP fix](docs/sessions/2026-04-13_cert-netpol-clusterip-fix.md)
- [Архив: первичная установка кластера](docs/archive/2026-03-05_initial-cluster-setup-guide.md)
- [Деплой кластера на новый VPS](docs/cluster-deploy-on-new-vps.md)
- [Установка ArgoCD](docs/argocd-setup.md)

## Известные TODO

- `alertmanager-sealed-secret.yaml` — placeholder: запечатать реальный
  конфиг ключом кластерного sealed-secrets контроллера (после ротации
  Telegram-токена).
- Версии Calico и ArgoCD не зафиксированы в манифестах — закрепить.
- `infrastructure/cert-manager/cluster_issuer.yml` — подставить реальный
  контактный email для Let's Encrypt.
- Образы Prometheus/Grafana/Alertmanager/node-exporter запинить вместо
  `latest`.

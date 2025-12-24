## Maintenance / Deployment Diagram — обслуживание (CI/CD → test → stage → prod + monitoring + rollback)

```mermaid
flowchart LR

subgraph DEV["Development"]
  DEVPC["Developer PC"]
end

REPO["GitHub Repo"]

subgraph PIPE["CI CD GitHub Actions"]
  CI1["CI Lint"]
  CI2["CI Tests"]
  CI3["Build Images"]
  CI4["Security Scan"]
end

REG["Container Registry"]

subgraph TEST["Test Environment"]
  TESTNS["K8s Namespace Test"]
  QA["QA Manual"]
  AT["Auto Tests"]
end

STAGE["Stage Environment"]

subgraph PROD["Production"]
  PRODK8S["Kubernetes Prod"]
end

subgraph OBS["Monitoring and Alerts"]
  PROM["Prometheus"]
  GRAF["Grafana"]
  LOGS["Logs Loki or ELK"]
  ALERT["Alertmanager"]
end

DEVPC --> REPO
REPO --> CI1
CI1 --> CI2
CI2 --> CI3
CI3 --> CI4
CI4 --> REG

REG --> TESTNS
TESTNS --> QA
TESTNS --> AT

TESTNS --> STAGE
STAGE --> PRODK8S

TESTNS --> PROM
STAGE --> PROM
PRODK8S --> PROM
PRODK8S --> LOGS

PROM --> GRAF
PROM --> ALERT

ALERT --> PRODK8S
QA --> PRODK8S

REG --> DEVPC

```

#### Maintenance / Deployment — обслуживание

Диаграмма обслуживания показывает **жизненный цикл изменений**: от коммита разработчика до обновления production, а также мониторинг и откаты.

1. **Разработка**

- Разработчик делает изменения локально и отправляет их в **GitHub** (push + Pull Request).
- В PR выполняется code review.

2. **CI/CD (GitHub Actions)**
   После push/merge запускаются workflow:

- **Lint** (проверка стиля и статический анализ),
- **Tests** (unit/integration, при наличии — e2e),
- **Build Images** (сборка Docker-образов сервисов),
- **Security Scan** (проверка уязвимостей образов/зависимостей).

Артефакты публикуются в **Container Registry**. Рекомендуемая практика — тегировать образы как минимум:

- `service:sha-<commit>` (точная воспроизводимость),
- `service:vX.Y.Z` (релизные теги),
  а `latest` использовать только как удобный алиас.

3. **Окружения (Test → Stage → Prod)**

- **Test**: развертывание в тестовый namespace, прогон smoke/regression, ручная проверка QA.
- **Stage**: финальная проверка конфигураций и интеграций, приближено к production.
- **Prod**: развертывание в production Kubernetes.

Стратегия выката на production — **Rolling Update**: новые pod’ы поднимаются постепенно, затем старые выключаются. Это позволяет обновляться без простоя при корректно настроенных readiness/liveness probes.

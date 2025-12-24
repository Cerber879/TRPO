## Infrastructure / Deployment Diagram — Система поиска и рекомендаций ресторанов

```mermaid
flowchart LR

subgraph INTERNET["Public Internet"]
  DNS_I["DNS"]
  WEB_I["Web SPA React"]
  MOB_I["Mobile App React Native"]
end

subgraph EXT["External Services"]
  MAPS_E["Yandex Maps API"]
  OAUTH_E["OAuth Providers"]
  MAIL_E["Email Provider"]
end

subgraph K8S["Kubernetes Cluster"]
  direction TB

  subgraph EDGE["Gateway Layer"]
    LB_K["Load Balancer"]
    ING_K["NGINX Ingress"]
    GW_K["API Gateway"]
  end

  subgraph APP["Application Deployments"]
    AUTH_S["Auth Service"]
    REST_S["Restaurant Service"]
    SEARCH_S["Search Service"]
    RECO_S["Recommendations Service"]
    REV_S["Reviews Service"]
    MOD_S["Moderation Service"]
    ADS_S["Advertising Service"]
    AN_S["Analytics Service"]
    IMG_S["Image Service"]
    WEB_S["Web App Nginx"]
  end

  subgraph INFRA["Cluster Infra"]
    RAB_K["RabbitMQ"]
    RED_K["Redis"]
  end

  subgraph OBS["Monitoring"]
    PROM_K["Prometheus"]
    GRAF_K["Grafana"]
    LOKI_K["Loki or ELK"]
  end
end

subgraph DATA["Data Layer"]
  PG_D["PostgreSQL Cluster"]
  IMG_D["Image Storage S3 or FS"]
end

WEB_I --> DNS_I
MOB_I --> DNS_I

DNS_I --> LB_K
LB_K --> ING_K
ING_K --> GW_K

GW_K --> AUTH_S
GW_K --> REST_S
GW_K --> SEARCH_S
GW_K --> RECO_S
GW_K --> REV_S
GW_K --> MOD_S
GW_K --> ADS_S
GW_K --> AN_S
GW_K --> IMG_S

WEB_I --> LB_K
MOB_I --> LB_K

APP --> RAB_K
APP --> RED_K
APP --> PG_D

IMG_S --> IMG_D

SEARCH_S -.-> MAPS_E
AUTH_S -.-> OAUTH_E
AUTH_S -.-> MAIL_E

APP --> PROM_K
RAB_K --> PROM_K
RED_K --> PROM_K

PROM_K --> GRAF_K
APP --> LOKI_K

```

### Пояснение к диаграммам Infrastructure / Deployment и Maintenance / Deployment

#### Infrastructure / Deployment — развертывание

Диаграмма показывает, **где и как физически развернута система** и как проходит пользовательский трафик.  
Пользователи работают через **Web SPA (React)** и **Mobile App (React Native)**. Запросы идут по HTTPS через **DNS → Load Balancer → NGINX Ingress**, затем попадают в **API Gateway**, который выполняет маршрутизацию запросов, проверку авторизации и ограничения (rate limiting).

Внутри Kubernetes развернуты микросервисы:

- Auth, Restaurant, Search, Recommendations, Reviews, Moderation, Advertising, Analytics, Image,
  а также компонент раздачи фронтенда (**Web App Nginx**).

Сервисы используют инфраструктурные компоненты кластера:

- **RabbitMQ** — обмен событиями и асинхронные задачи (event-driven взаимодействие),
- **Redis** — кэш и хранение сессионных данных (refresh tokens / sessions / быстрые кеши).

Хранение данных представлено отдельным слоем:

- **PostgreSQL Cluster** — основной datastore системы,
- **Image Storage (S3/FS)** — файловое хранилище изображений.

> Важно про БД “для каждого сервиса”: логически каждый сервис владеет своим набором данных (например, отдельные схемы/таблицы/учетные записи доступа — _schema-per-service_). Физически в учебном проекте эти данные могут располагаться в одном кластере PostgreSQL, чтобы не усложнять инфраструктуру. При необходимости это может быть расширено до _database-per-service_ (отдельные БД/инстансы).

Внешние интеграции:

- **Search Service** обращается к **Yandex Maps API** (геокодирование/поиск поблизости),
- **Auth Service** использует **OAuth Providers** (социальная авторизация) и **Email Provider** (верификация email/уведомления).

Наблюдаемость:

- **Prometheus** собирает метрики с приложений и инфраструктуры (через exporter’ы для Redis/RabbitMQ),
- **Grafana** визуализирует метрики,
- **Loki/ELK** агрегирует логи сервисов (как централизованное логирование).

Диаграмма агрегирует часть связей (например, “все сервисы → PostgreSQL/Redis/RabbitMQ”), чтобы избежать перегруженности стрелками. Детальные сервис-к-сервису взаимодействия (конкретные сценарии) обычно показываются на sequence/компонентных диаграммах.

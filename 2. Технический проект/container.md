# Container diagram (C4-like): Система поиска и рекомендаций ресторанов

> Примечания:
>
> - Sync вызовы: HTTP через API Gateway
> - Events: AMQP (RabbitMQ)
> - Cache/Sessions: Redis (TCP)
> - Images: File Storage (FS)

```mermaid
flowchart LR

  %% Actors
  visitor([Посетитель])
  owner([Владелец])
  moderator([Модератор])
  advertiser([Рекламодатель])

  %% System boundary
  subgraph SYS[Система поиска и рекомендаций ресторанов]
    web[Web app React]
    mobile[Mobile app React Native]
    gw[API Gateway Spring Cloud Gateway]

    rabbit[(RabbitMQ)]
    redis[(Redis)]

    %% Services grouped
    subgraph SVC[Микросервисы]
      auth[Auth Service JWT/OAuth/Session]
      rest[Restaurant Service]
      search[Search Service PostgreSQL FTS]
      reco[Recommendations Service Hybrid]
      reviews[Reviews Service]
      mod[Moderation Service]
      ads[Advertising Service]
      anal[Analytics Service]
      img[Image Service]
    end

    %% Per-service DB schemas (logical)
    authdb[(Auth DB schema)]
    restdb[(Restaurant DB schema)]
    searchdb[(Search DB schema)]
    recodb[(Reco DB schema)]
    reviewsdb[(Reviews DB schema)]
    moddb[(Moderation DB schema)]
    adsdb[(Ads DB schema)]
    analdb[(Analytics DB schema)]
    imgdb[(Image DB schema)]
    fs[(Image Storage File System)]
  end

  %% External systems
  maps[[Yandex Maps API]]
  oauth[[OAuth Providers]]

  %% Users -> Apps
  visitor --> web
  visitor --> mobile
  owner --> web
  moderator --> web
  advertiser --> web

  %% Apps -> Gateway
  web --> gw
  mobile --> gw

  %% Gateway -> Services (HTTP)
  gw -->|HTTP| auth
  gw -->|HTTP| rest
  gw -->|HTTP| search
  gw -->|HTTP| reco
  gw -->|HTTP| reviews
  gw -->|HTTP| mod
  gw -->|HTTP| ads
  gw -->|HTTP| anal
  gw -->|HTTP| img

  %% Service -> own DB schema
  auth -->|JDBC| authdb
  rest -->|JDBC| restdb
  search -->|JDBC| searchdb
  reco -->|JDBC| recodb
  reviews -->|JDBC| reviewsdb
  mod -->|JDBC| moddb
  ads -->|JDBC| adsdb
  anal -->|JDBC| analdb
  img -->|JDBC| imgdb
  img -->|FS| fs

  %% Infrastructure: events + cache
  gw -->|AMQP| rabbit
  rest -->|AMQP| rabbit
  reviews -->|AMQP| rabbit
  mod -->|AMQP| rabbit
  reco -->|AMQP| rabbit
  anal -->|AMQP| rabbit
  img -->|AMQP| rabbit

  auth -->|TCP| redis
  search -->|TCP| redis
  rest -->|TCP| redis
  reco -->|TCP| redis
  reviews -->|TCP| redis
  ads -->|TCP| redis

  %% External deps
  auth -->|HTTPS| oauth
  search -->|HTTPS| maps
```

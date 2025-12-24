# Диаграммы последовательности системы поиска и рекомендаций ресторанов

## 1) Регистрация пользователя

```mermaid
sequenceDiagram
    actor User as Пользователь
    participant UI as Клиент (Web/Mobile)
    participant API as API Gateway
    participant Auth as Auth Service
    participant UDB as Users DB (PostgreSQL)
    participant ES as Email Service
    participant Bus as Event Bus (RabbitMQ)
    participant NS as Notification Service
    participant Redis as Redis (Sessions)

    User->>UI: Заполнить форму регистрации
    UI->>API: POST /auth/register (email, password, role)
    API->>Auth: Register(cmd)
    Auth->>UDB: Проверка email + создание пользователя (status=PENDING)
    alt Email уже существует
        UDB-->>Auth: Conflict
        Auth-->>API: 409 EmailExists
        API-->>UI: Ошибка "email уже занят"
    else Успешно
        UDB-->>Auth: userId
        Auth->>ES: SendVerification(email, token)
        Auth->>Bus: Event UserRegistered(userId, role)
        Bus-->>NS: Deliver UserRegistered
        NS-->>UI: Push "Проверьте email"
        Auth-->>API: 201 Created
        API-->>UI: Регистрация успешна (ожидается подтверждение)
    end

    User->>UI: Перейти по ссылке из письма
    UI->>API: GET /auth/verify?token=...
    API->>Auth: VerifyEmail(token)
    Auth->>UDB: Update user status=ACTIVE
    UDB-->>Auth: OK
    Auth->>Bus: Event EmailVerified(userId)
    Auth-->>API: 200 OK
    API-->>UI: Email подтверждён, можно войти
```

## 2) Аутентификация через OAuth

```mermaid
sequenceDiagram
    actor User as Пользователь
    participant UI as Клиент (Web/Mobile)
    participant API as API Gateway
    participant Auth as Auth Service
    participant OAuth as OAuth Provider (Google/Facebook)
    participant UDB as Users DB (PostgreSQL)
    participant Redis as Redis (Sessions/Tokens)

    User->>UI: Войти через Google/Facebook
    UI->>API: GET /auth/oauth/google
    API->>Auth: InitiateOAuth(provider)
    Auth->>OAuth: Redirect to OAuth provider
    OAuth-->>UI: OAuth consent screen
    User->>OAuth: Разрешить доступ
    OAuth-->>UI: Callback с кодом авторизации
    UI->>API: GET /auth/oauth/callback?code=...
    API->>Auth: OAuthCallback(code, provider)
    Auth->>OAuth: Exchange code for token
    OAuth-->>Auth: Access token + user info
    Auth->>UDB: Find or create user by OAuth ID
    UDB-->>Auth: userId
    Auth->>Auth: Generate JWT tokens (access + refresh)
    Auth->>Redis: Store refresh token
    Auth-->>API: JWT tokens
    API-->>UI: 200 OK + tokens
    UI-->>User: Успешная авторизация
```

## 3) Поиск ресторанов

```mermaid
sequenceDiagram
    actor User as Посетитель
    participant UI as Клиент (Web/Mobile)
    participant API as API Gateway
    participant Auth as Auth Service
    participant Search as Search Service
    participant Redis as Redis (Cache)
    participant DB as PostgreSQL
    participant RS as Restaurant Service
    participant Maps as Yandex Maps API

    User->>UI: Ввести поисковый запрос (название, кухня, местоположение)
    UI->>API: GET /restaurants/search?query=...&location=...
    API->>Auth: Проверка JWT (опционально)
    Auth-->>API: OK
    API->>Search: SearchRestaurants(query, filters, location)
    
    Search->>Redis: Проверить кэш результатов
    alt Результаты в кэше
        Redis-->>Search: Кэшированные результаты
        Search-->>API: Results
        API-->>UI: 200 OK + список ресторанов
    else Кэш отсутствует
        Search->>DB: Полнотекстовый поиск (PostgreSQL Full-Text Search)
        Search->>Maps: Геокодирование адреса / поиск поблизости
        Maps-->>Search: Координаты / список ресторанов в радиусе
        Search->>RS: Получить детали ресторанов
        RS-->>Search: Данные ресторанов
        Search->>DB: Применить фильтры (рейтинг, цена, кухня)
        DB-->>Search: Отфильтрованные результаты
        Search->>Redis: Кэшировать результаты (TTL)
        Search-->>API: Results
        API-->>UI: 200 OK + список ресторанов
    end
```

## 4) Просмотр карточки ресторана

```mermaid
sequenceDiagram
    actor User as Посетитель
    participant UI as Клиент (Web/Mobile)
    participant API as API Gateway
    participant Auth as Auth Service
    participant RS as Restaurant Service
    participant Redis as Redis (Cache)
    participant DB as PostgreSQL
    participant Reviews as Reviews Service
    participant Images as Image Service
    participant Analytics as Analytics Service
    participant Bus as Event Bus (RabbitMQ)

    User->>UI: Открыть карточку ресторана
    UI->>API: GET /restaurants/{id}
    API->>Auth: Проверка JWT (опционально)
    Auth-->>API: userId (если авторизован)
    API->>RS: GetRestaurant(id, userId)
    
    RS->>Redis: Проверить кэш ресторана
    alt Ресторан в кэше
        Redis-->>RS: Кэшированные данные
    else Кэш отсутствует
        RS->>DB: SELECT restaurant data
        DB-->>RS: Данные ресторана
        RS->>Reviews: GetRating(restaurantId)
        Reviews-->>RS: Средний рейтинг, количество отзывов
        RS->>Images: GetImages(restaurantId)
        Images-->>RS: Список изображений
        RS->>Redis: Кэшировать данные ресторана
    end
    
    RS-->>API: RestaurantDTO (полные данные)
    API->>Analytics: TrackView(restaurantId, userId)
    Analytics->>Bus: Event RestaurantViewed(restaurantId, userId, timestamp)
    API-->>UI: 200 OK + RestaurantDTO
    UI-->>User: Отображение карточки ресторана
```

## 5) Создание отзыва и оценки

```mermaid
sequenceDiagram
    actor User as Посетитель
    participant UI as Клиент (Web/Mobile)
    participant API as API Gateway
    participant Auth as Auth Service
    participant Reviews as Reviews Service
    participant RS as Restaurant Service
    participant DB as PostgreSQL
    participant Redis as Redis (Cache)
    participant Bus as Event Bus (RabbitMQ)
    participant Moderation as Moderation Service
    participant Analytics as Analytics Service
    participant NS as Notification Service

    User->>UI: Написать отзыв и поставить оценку
    UI->>API: POST /reviews (restaurantId, rating, text)
    API->>Auth: Проверка JWT + валидация роли
    Auth-->>API: userId, role=VISITOR
    API->>Reviews: CreateReview(cmd: restaurantId, userId, rating, text)
    
    Reviews->>DB: Проверка: отзыв от этого пользователя уже существует?
    alt Отзыв уже существует
        DB-->>Reviews: Conflict
        Reviews-->>API: 409 ReviewAlreadyExists
        API-->>UI: Ошибка "Вы уже оставили отзыв"
    else Новый отзыв
        Reviews->>Moderation: CheckContent(text)
        alt Контент подозрительный
            Moderation-->>Reviews: FLAGGED
            Reviews->>DB: Save Review(status=ON_REVIEW, rating, text)
            DB-->>Reviews: reviewId
            Reviews->>Bus: Event ReviewCreated(reviewId, status=ON_REVIEW)
            Reviews-->>API: 202 Accepted (на модерации)
            API-->>UI: Отзыв отправлен на проверку
        else ОК
            Moderation-->>Reviews: OK
            Reviews->>DB: Save Review(status=APPROVED, rating, text)
            DB-->>Reviews: reviewId
            Reviews->>Redis: Инвалидировать кэш рейтинга ресторана
            Reviews->>Bus: Event ReviewCreated(reviewId, status=APPROVED, rating)
            
            Bus-->>RS: Consume ReviewCreated
            RS->>DB: Пересчитать средний рейтинг ресторана
            RS->>Redis: Обновить кэш рейтинга
            RS->>Bus: Event RestaurantRatingUpdated(restaurantId, newRating)
            
            Bus-->>Analytics: Consume ReviewCreated
            Analytics->>DB: Записать событие для аналитики
            
            Bus-->>NS: Consume ReviewCreated
            NS-->>UI: Push "Отзыв опубликован"
            
            Reviews-->>API: 201 Created (reviewId)
            API-->>UI: Отзыв успешно создан
        end
    end
```

## 6) Получение персонализированных рекомендаций

```mermaid
sequenceDiagram
    actor User as Посетитель
    participant UI as Клиент (Web/Mobile)
    participant API as API Gateway
    participant Auth as Auth Service
    participant Rec as Recommendations Service
    participant Redis as Redis (Cache)
    participant DB as PostgreSQL
    participant RS as Restaurant Service
    participant Reviews as Reviews Service
    participant Bus as Event Bus (RabbitMQ)

    User->>UI: Открыть главную страницу
    UI->>API: GET /recommendations
    API->>Auth: Проверка JWT
    Auth-->>API: userId
    API->>Rec: GetRecommendations(userId)
    
    Rec->>Redis: Проверить кэш рекомендаций для userId
    alt Рекомендации в кэше (не старше 1 часа)
        Redis-->>Rec: Кэшированные рекомендации
        Rec-->>API: Recommendations
        API-->>UI: 200 OK + список рекомендованных ресторанов
    else Кэш отсутствует или устарел
        Rec->>DB: Получить историю действий пользователя (просмотры, отзывы, поиски)
        DB-->>Rec: User history
        Rec->>Reviews: GetUserRatings(userId)
        Reviews-->>Rec: Предпочтения пользователя (любимые типы кухни, ценовые категории)
        Rec->>RS: GetRestaurants(filters based on preferences)
        RS-->>Rec: Список ресторанов
        
        note over Rec: Применение алгоритмов рекомендаций:<br/>- Content-Based Filtering<br/>- Collaborative Filtering<br/>- Гибридный подход
        
        Rec->>DB: Рассчитать релевантность для каждого ресторана
        DB-->>Rec: Ранжированный список
        Rec->>Redis: Кэшировать рекомендации (TTL=1 час)
        Rec-->>API: Recommendations (топ-10 ресторанов)
        API-->>UI: 200 OK + список рекомендованных ресторанов
    end
    
    UI->>User: Отображение рекомендаций
    
    note over Bus: В фоне: события UserAction<br/>обновляют модель рекомендаций
```

## 7) Создание ресторана владельцем

```mermaid
sequenceDiagram
    actor Owner as Владелец ресторана
    participant UI as Клиент (Web/Mobile)
    participant API as API Gateway
    participant Auth as Auth Service
    participant RS as Restaurant Service
    participant DB as PostgreSQL
    participant Redis as Redis (Cache)
    participant Images as Image Service
    participant Bus as Event Bus (RabbitMQ)
    participant Moderation as Moderation Service
    participant Search as Search Service
    participant NS as Notification Service

    Owner->>UI: Заполнить форму создания ресторана
    UI->>API: POST /restaurants (name, description, address, cuisine, priceRange, photos)
    API->>Auth: Проверка JWT + валидация роли=OWNER
    Auth-->>API: userId, role=OWNER
    API->>RS: CreateRestaurant(cmd: ownerId, data, photos)
    
    RS->>DB: Проверка: ресторан с таким названием/адресом уже существует?
    alt Ресторан уже существует
        DB-->>RS: Conflict
        RS-->>API: 409 RestaurantAlreadyExists
        API-->>UI: Ошибка "Ресторан уже зарегистрирован"
    else Новый ресторан
        RS->>Images: UploadImages(photos)
        Images-->>RS: Image URLs
        RS->>Moderation: CheckContent(description, name)
        alt Контент подозрительный
            Moderation-->>RS: FLAGGED
            RS->>DB: Save Restaurant(status=ON_REVIEW, ownerId, data)
            DB-->>RS: restaurantId
            RS->>Bus: Event RestaurantCreated(restaurantId, status=ON_REVIEW)
            RS-->>API: 202 Accepted (на модерации)
            API-->>UI: Ресторан отправлен на проверку
        else ОК
            Moderation-->>RS: OK
            RS->>DB: Save Restaurant(status=APPROVED, ownerId, data, imageUrls)
            DB-->>RS: restaurantId
            RS->>Redis: Инвалидировать кэш списков ресторанов
            RS->>Bus: Event RestaurantCreated(restaurantId, status=APPROVED)
            
            Bus-->>Search: Consume RestaurantCreated
            Search->>DB: Индексировать ресторан для полнотекстового поиска
            Search->>Redis: Инвалидировать кэш поиска
            
            Bus-->>NS: Consume RestaurantCreated
            NS-->>UI: Push "Ресторан опубликован"
            
            RS-->>API: 201 Created (restaurantId)
            API-->>UI: Ресторан успешно создан
        end
    end
```

## 8) Модерация отзыва модератором

```mermaid
sequenceDiagram
    actor Moderator as Модератор
    participant UI as Клиент (Web)
    participant API as API Gateway
    participant Auth as Auth Service
    participant Moderation as Moderation Service
    participant Reviews as Reviews Service
    participant RS as Restaurant Service
    participant DB as PostgreSQL
    participant Redis as Redis (Cache)
    participant Bus as Event Bus (RabbitMQ)
    participant NS as Notification Service

    Moderator->>UI: Открыть очередь модерации
    UI->>API: GET /moderation/queue
    API->>Auth: Проверка JWT + валидация роли=MODERATOR
    Auth-->>API: userId, role=MODERATOR
    API->>Moderation: GetModerationQueue()
    Moderation->>DB: SELECT items with status=ON_REVIEW
    DB-->>Moderation: Список элементов на модерации
    Moderation-->>API: ModerationQueue
    API-->>UI: 200 OK + список элементов на модерации

    Moderator->>UI: Одобрить/отклонить отзыв
    UI->>API: POST /moderation/reviews/{id}/approve или /reject
    API->>Moderation: ModerateReview(reviewId, action, reason)
    Moderation->>DB: Update Review(status=APPROVED/REJECTED, moderatorId, timestamp)
    DB-->>Moderation: OK
    
    alt Отзыв одобрен
        Moderation->>Reviews: ReviewApproved(reviewId)
        Reviews->>Redis: Инвалидировать кэш рейтинга ресторана
        Reviews->>Bus: Event ReviewApproved(reviewId, rating)
        Bus-->>RS: Consume ReviewApproved
        RS->>DB: Пересчитать средний рейтинг ресторана
        RS->>Redis: Обновить кэш рейтинга
        Bus-->>NS: Consume ReviewApproved
        NS-->>UI: Push автору "Ваш отзыв опубликован"
    else Отзыв отклонён
        Moderation->>Reviews: ReviewRejected(reviewId, reason)
        Reviews->>Bus: Event ReviewRejected(reviewId, reason)
        Bus-->>NS: Consume ReviewRejected
        NS-->>UI: Push автору "Отзыв отклонён: {reason}"
    end
    
    Moderation-->>API: 200 OK
    API-->>UI: Модерация завершена
```

## 9) Просмотр аналитики владельцем ресторана

```mermaid
sequenceDiagram
    actor Owner as Владелец ресторана
    participant UI as Клиент (Web/Mobile)
    participant API as API Gateway
    participant Auth as Auth Service
    participant Analytics as Analytics Service
    participant RS as Restaurant Service
    participant Reviews as Reviews Service
    participant DB as PostgreSQL
    participant Redis as Redis (Cache)

    Owner->>UI: Открыть страницу аналитики
    UI->>API: GET /analytics/restaurants/{id}?period=...
    API->>Auth: Проверка JWT + валидация роли=OWNER + проверка владения рестораном
    Auth-->>API: userId, role=OWNER
    API->>RS: CheckOwnership(restaurantId, userId)
    RS-->>API: OK (пользователь является владельцем)
    
    API->>Analytics: GetAnalytics(restaurantId, period)
    Analytics->>Redis: Проверить кэш аналитики
    alt Аналитика в кэше (не старше 5 минут)
        Redis-->>Analytics: Кэшированные данные
    else Кэш отсутствует
        Analytics->>DB: Агрегация: количество просмотров карточки
        DB-->>Analytics: View count
        Analytics->>DB: Агрегация: количество кликов на контакты
        DB-->>Analytics: Click count
        Analytics->>Reviews: GetRatingStats(restaurantId, period)
        Reviews-->>Analytics: Рейтинг, количество отзывов, динамика
        Analytics->>DB: Агрегация: конверсия рекомендаций
        DB-->>Analytics: Conversion rate
        Analytics->>Redis: Кэшировать аналитику (TTL=5 минут)
    end
    
    Analytics-->>API: AnalyticsDTO (просмотры, клики, рейтинг, отзывы, конверсия)
    API-->>UI: 200 OK + AnalyticsDTO
    UI-->>Owner: Отображение графиков и статистики
```

## 10) Создание рекламной кампании

```mermaid
sequenceDiagram
    actor Advertiser as Рекламодатель
    participant UI as Клиент (Web/Mobile)
    participant API as API Gateway
    participant Auth as Auth Service
    participant Advertising as Advertising Service
    participant RS as Restaurant Service
    participant DB as PostgreSQL
    participant Redis as Redis (Cache)
    participant Bus as Event Bus (RabbitMQ)

    Advertiser->>UI: Создать рекламную кампанию
    UI->>API: POST /advertising/campaigns (restaurantId, creative, targetAudience, budget, period)
    API->>Auth: Проверка JWT + валидация роли=ADVERTISER или OWNER
    Auth-->>API: userId, role
    API->>Advertising: CreateCampaign(cmd)
    
    Advertising->>RS: VerifyRestaurantExists(restaurantId)
    RS-->>Advertising: OK
    Advertising->>DB: Save Campaign(status=ACTIVE, restaurantId, creative, targeting, budget, startDate, endDate)
    DB-->>Advertising: campaignId
    Advertising->>Redis: Кэшировать активную рекламу
    Advertising->>Bus: Event CampaignCreated(campaignId, restaurantId)
    Advertising-->>API: 201 Created (campaignId)
    API-->>UI: Рекламная кампания создана
    
    note over Bus: Кампания будет показываться<br/>в поиске и рекомендациях<br/>согласно таргетингу
```


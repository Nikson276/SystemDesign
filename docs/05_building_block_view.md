---
title: "Вид строительных блоков (микросервисы)"
status: draft
version: 0.1
author: "Михайлов Никита"
date: 2025-11-06
tags: [микросервисы, DDD, C4, контейнеры, Kafka]
---

# Вид строительных блоков (микросервисы)

Система ТУНЕЦ разбита на микросервисы в соответствии с принципами **Domain-Driven Design**. Каждый микросервис — автономный «контейнер» с собственной логикой, хранилищем и API.

## Диаграмма контейнеров (C4 Level 2)

<details>
<summary>Диаграмма C2</summary>

![C2 level](_media/c2_level.png)

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

Person(user, "Пользователь", "Слушает, ищет, создаёт плейлисты")
Person(artist, "Автор", "Загружает треки")
Person(admin, "Администратор", "Управляет контентом, обновляет метаданные")

System_Boundary(tunec, "ТУНЕЦ") {

    Container(api_gateway, "API Gateway", "Go", "Маршрутизация запросов, rate limiting, делегирование auth в Auth Service")

    Container_Boundary(auth_block, "Auth & Subscription Layer") {
        Container(auth, "Auth Service", "Go/Python", "Регистрация, логин, OAuth, JWT/Refresh, валидация токена")
        Container(subscription, "Subscription Service", "Python", "Управление подписками, шаги SAGA")
        Container(notification, "Notification Service", "Python", "Email и push уведомления на события")
        Container(payments, "Payments Service", "Python", "Обработка платежей через СБП и USDT")
    }

    Container_Boundary(content_block, "Content Layer") {
        Container(catalog, "Catalog Service", "Python", "Метаданные треков, альбомы, исполнители (CQRS Read side)")
        Container(ingestion, "Ingestion Service", "Go", "Загрузка и валидация аудиофайлов от авторов (CQRS Write side)")
        Container(cdp, "CDP Service", "Java/Debezium", "Change Data Processing — отслеживание изменений в Postgres и публикация событий в Kafka")
        Container(indexer, "Catalog Indexer Service", "Python", "Потребляет события треков, обновляет Elasticsearch и инвалидирует Redis")
    }

    Container_Boundary(playback_block, "Playback Layer") {
        Container(playback, "Playback Service", "Go", "Выдача CDN ссылок треков, логирование PlaybackStarted")
    }

    Container_Boundary(user_block, "User & Engagement Layer") {
        Container(users, "User Service", "Python", "Профили пользователей, плейлисты")
        Container(recommendations, "Recommendations Service", "Python", "Рекомендации на основе истории прослушиваний")
        Container(ads, "Ads Service", "Go", "Реклама для free-пользователей")
    }

    ContainerDb(pg, "PostgreSQL", "Write модель: каталог, пользователи, подписки, платежи")
    ContainerDb(es, "Elasticsearch", "Read модель: поиск по каталогу")
    ContainerDb(s3, "Object Storage (S3)", "Хранение аудиофайлов")
    ContainerDb(redis, "Redis", "Кэш JWT, is_premium, горячие выборки поиска, rate limiting")
    Container(kafka, "Kafka", "Event backbone: TrackPublished, PaymentCompleted, SubscriptionActivated, UserUpdated")
}

System(google_oauth, "Google OAuth", "Внешняя авторизация OAuth2")
System(vk_oauth, "VK OAuth", "Внешняя авторизация OAuth2")
System(yandex_oauth, "Yandex OAuth", "Внешняя авторизация OAuth2")
System(sbp, "СБП", "Платежи")
System(usdt, "USDT", "Криптоплатежи")
System(debezium, "Debezium", "CDC listener для Postgres")
System(events_stats, "Events Statistics Service", "Сбор аналитики воспроизведений и поведения пользователей")

Rel(kafka, events_stats, "Consume", "PlaybackStarted, PlaybackFinished")

Rel(user, api_gateway, "HTTPS", "Login/Register, запросы с JWT")
Rel(artist, api_gateway, "HTTPS", "Загрузка треков (через Ingestion)")
Rel(admin, pg, "SQL/ORM", "Изменение метаданных напрямую или через админку")

Rel(api_gateway, auth, "HTTP", "/login, /register, /refresh")
Rel(auth, google_oauth, "OAuth2/OpenID Connect")
Rel(auth, vk_oauth, "OAuth2/OpenID Connect")
Rel(auth, yandex_oauth, "OAuth2/OpenID Connect")
Rel(auth, redis, "Set/Get", "Кэш токенов и claims")
Rel(api_gateway, redis, "GET", "Проверка токена и флага is_premium")

Rel(api_gateway, catalog, "HTTP", "GET /tracks, поиск")
Rel(api_gateway, ingestion, "HTTP", "POST /tracks")
Rel(api_gateway, playback, "HTTP", "GET /stream/{id}")
Rel(api_gateway, users, "HTTP", "GET /playlists")
Rel(api_gateway, ads, "HTTP", "GET /ad")

Rel(api_gateway, payments, "HTTP", "POST /subscribe")
Rel(payments, kafka, "Publish", "PaymentCompleted / PaymentFailed")
Rel(kafka, subscription, "Consume", "PaymentCompleted → SubscriptionActivated; PaymentFailed → стоп процесса")
Rel(subscription, kafka, "Publish", "SubscriptionActivated / SubscriptionFailed")
Rel(kafka, notification, "Consume", "SubscriptionActivated → письмо пользователю")
Rel(kafka, users, "Consume", "SubscriptionActivated → обновление is_premium")
Rel(users, kafka, "Publish", "UserUpdated (is_premium)")
Rel(kafka, auth, "Consume", "UserUpdated → обновление JWT/Refresh")

Rel(ingestion, kafka, "Publish", "TrackPublished")
Rel(playback, kafka, "Publish", "PlaybackStarted")

Rel(kafka, catalog, "Consume", "TrackPublished → сохранение метаданных")
Rel(kafka, recommendations, "Consume", "PlaybackStarted, PaymentCompleted")

BiRel(catalog, pg, "Чтение/запись")
BiRel(users, pg, "Чтение/запись")
BiRel(subscription, pg, "Чтение/запись")
BiRel(payments, pg, "Чтение/запись")

Rel(catalog, es, "Batch sync", "Индекс поиска")
Rel(playback, s3, "Чтение аудиофайлов", "Pre-signed URL / CDN")
Rel(api_gateway, redis, "Кэш JWT/is_premium", "GET/SET")

Rel(cdp, debezium, "Подписка на WAL", "Logical replication")
Rel(debezium, cdp, "CDC Events", "TrackUpdated/TrackDeleted")
Rel(cdp, kafka, "Publish", "CDC-based events для индексации")
Rel(kafka, indexer, "Consume", "TrackPublished, TrackUpdated, TrackDeleted")
Rel(indexer, es, "Update index", "Добавление/обновление треков")
Rel(indexer, redis, "Invalidate cache", "По ключам треков и поисковым запросам")

Rel(payments, sbp, "Open API", "Инициация платежа")
Rel(payments, usdt, "Webhook + API", "Подтверждение транзакции")

@enduml
```

</details>

## 📦 Описание контейнеров

|Контейнер/Система|Ответственность|Технологии|Хранилище / Интеграции|
|---|---|---|---|
|**API Gateway**|Единая точка входа: аутентификация, маршрутизация, rate limiting, проверка флагов `is_premium`|Go|Redis (кэш сессий, токенов, `is_premium`)|
|**Ingestion Service**|Приём, валидация и транскодирование аудиофайлов от авторов (CQRS Write side)|Go|S3 (временное хранение → каталог), PostgreSQL (метаданные), Kafka (`TrackPublished`)|
|**Catalog Service**|Управление метаданными треков, альбомов, исполнителей (CQRS Read side)|Python|PostgreSQL (источник истины), Elasticsearch (поисковый индекс)|
|**CDP Service**|Change Data Processing: ловит изменения в PostgreSQL (Debezium CDC) и отправляет события в Kafka|Java/Debezium|PostgreSQL (WAL-репликация → CDC Events)|
|**Catalog Indexer Service**|Консюмит события треков (TrackPublished, TrackUpdated), обновляет Elasticsearch, инвалидирует Redis|Python|Elasticsearch, Redis (инвалидация кэшей поиска)|
|**Playback Service**|Выдача пред‑подписанных URL к аудиофайлам через CDN, логирование событий воспроизведения|Go|S3/CDN, Kafka (`PlaybackStarted`, `PlaybackFinished`)|
|**User Service**|Профили, плейлисты, импорт избранных треков из VK / Яндекс.Музыки|Python|PostgreSQL, Kafka (`UserUpdated`)|
|**Payments Service**|Обработка платежей/подписок, интеграция с СБП / USDT|Python|PostgreSQL, Kafka (`PaymentCompleted`, `PaymentFailed`)|
|**Subscription Service**|Логика подписок, пошаговая SAGA при активации/отказе|Python|PostgreSQL, Kafka (`SubscriptionActivated`, `SubscriptionFailed`)|
|**Notification Service**|Отправка Email и push-уведомлений на события|Python|— (провайдеры уведомлений)|
|**Ads Service**|Выдача рекламных ассетов пользователям без подписки|Go|—|
|**Recommendations Service**|Рекомендации на основе истории прослушиваний (batch jobs + события)|Python|PostgreSQL (история прослушиваний), Kafka (PlaybackStarted)|
|**Админка (UI + BFF)**|Веб-интерфейс для авторов и администраторов|TypeScript/React|—|
|**Events Statistics Service** 🔹|Внешний сервис аналитики воспроизведений — потребляет Playback события из Kafka|(внешний)|Собственное хранилище статистики (вне границ «Тунец»)|

---

## 📌 Ключевые принципы

> **Event‑Driven архитектура:** все ключевые бизнес‑события (`TrackPublished`, `TrackUpdated`, `PlaybackStarted`, `PaymentCompleted`) публикуются в Kafka; внешние системы (например, Events Statistics) получают доступ через потребление топиков.
> 
> **CQRS:** Write‑модель (PostgreSQL + Kafka) отделена от Read‑модели (Elasticsearch + Redis). Синхронизация поискового индекса выполняется асинхронно через Catalog Indexer и CDP Service (Debezium CDC).
> 
> **Источник истины:** PostgreSQL — единственное хранилище для транзакционных данных, Read‑модель всегда пересчитывается на его основе.
> 
> **Read‑оптимизация:** Elasticsearch используется только для поиска; Redis — для горячих выборок и хранения признаков (например, `is_premium`).
> 
> **Логика показа рекламы на клиенте:** флаг `is_premium` передаётся в JWT, плеер на стороне клиента решает, вставлять рекламу или нет.
> 
> **CDN готовность:** аудиофайлы хранятся в Object Storage (S3‑совместимом), доставляются через CDN;
> 
> **Внешняя аналитика:** события воспроизведения направляются в внешний Events Statistics Service, интеграция построена через Kafka, что обеспечивает надёжность и слабую связанность.

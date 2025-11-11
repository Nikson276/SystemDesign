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

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

Person(user, "Пользователь", "Слушает, ищет, создаёт плейлисты")
Person(artist, "Автор", "Загружает треки")

System_Boundary(tunec, "ТУНЕЦ") {

  Container(admin_ui, "Админка (UI + BFF)", "TypeScript/React", "Веб-интерфейс для загрузки треков. Вызывает Ingestion API.")
  Container(api_gateway, "API Gateway", "Go", "JWT-аутентификация, маршрутизация, rate limiting. Содержит флаг is_premium в токене.")

  Container(ingestion, "Ingestion Service", "Go", "Приём и валидация аудиофайлов от авторов. Публикует TrackPublished в Kafka.")
  Container(catalog, "Catalog Service", "Python", "Управление метаданными треков, альбомов, исполнителей. Хранит данные в PostgreSQL.")
  Container(playback, "Playback Service", "Go", "Выдаёт URL аудиофайлов (прокси через CDN). Логирует прослушивания.")
  Container(users, "Users Service", "Python", "Управление профилями, плейлистами, импортом из VK/Yandex.Музыки.")
  Container(payments, "Payments Service", "Python", "Обработка подписок через СБП и USDT. Обновляет статус is_premium.")
  Container(ads, "Ads Service", "Go", "Выдаёт рекламные ассеты каждые 7 минут для free-пользователей.")
  Container(recommendations, "Recommendations Service", "Python", "Генерирует рекомендации раз в сутки на основе истории прослушиваний.")

  ContainerDb(pg, "PostgreSQL", "Источник истины для: каталог, пользователи, плейлисты, подписки")
  ContainerDb(es, "Elasticsearch", "Индекс для full-text search с фильтрацией")
  ContainerDb(s3, "Object Storage (S3)", "Хранение аудиофайлов (MP3/FLAC)")
  ContainerDb(redis, "Redis", "Кэш: is_premium, сессии, rate limiting")

  Container(kafka, "Kafka", "Event backbone: TrackPublished, PaymentCompleted, PlaybackStarted и др.")
}

' Внешние зависимости
System(ym, "Яндекс.Музыка", "Импорт треков")
System(vk, "VK", "Импорт треков")
System(sbp, "СБП", "Платежи")
System(usdt, "USDT", "Криптоплатежи")

' Связи пользователя
Rel(user, api_gateway, "HTTPS", "Все запросы")
Rel(artist, admin_ui, "HTTPS", "Загрузка треков")

' Внутренние связи
Rel(admin_ui, ingestion, "HTTP", "POST /tracks")
Rel(api_gateway, catalog, "HTTP", "GET /tracks, /search")
Rel(api_gateway, users, "HTTP", "GET /playlists, POST /import")
Rel(api_gateway, payments, "HTTP", "POST /subscribe")
Rel(api_gateway, ads, "HTTP", "GET /ad")
Rel(api_gateway, playback, "HTTP", "GET /stream/{id}")

' Сервис → Kafka
Rel(ingestion, kafka, "Публикует", "TrackPublished")
Rel(payments, kafka, "Публикует", "PaymentCompleted")
Rel(playback, kafka, "Публикует", "PlaybackStarted")

' Kafka → подписчики
Rel(kafka, catalog, "Подписка", "TrackPublished → обогащение метаданных")
Rel(kafka, recommendations, "Подписка", "PlaybackStarted, PaymentCompleted")
Rel(kafka, users, "Подписка", "PaymentCompleted → обновление профиля")

' Хранилища
BiRel(catalog, pg, "Чтение/запись")
BiRel(users, pg, "Чтение/запись")
BiRel(payments, pg, "Чтение/запись")

Rel(catalog, es, "Полнотекстовый индекс", "Sync via batch")
Rel(playback, s3, "Чтение аудиофайлов", "Pre-signed URL / CDN")
Rel(api_gateway, redis, "Кэш JWT и is_premium", "GET/SET")

' Интеграции
Rel(users, ym, "OAuth + API", "Импорт избранного")
Rel(users, vk, "OAuth + API", "Импорт избранного")
Rel(payments, sbp, "Open API", "Инициация платежа")
Rel(payments, usdt, "Webhook + API", "Подтверждение транзакции")

@enduml
```

</details>

## Описание контейнеров

| Контейнер                  | Ответственность                                                                 | Технологии        | Хранилище                                      |
|---------------------------|----------------------------------------------------------------------------------|-------------------|------------------------------------------------|
| **API Gateway**           | Единая точка входа, аутентификация, маршрутизация, rate limiting                | Go                | Redis (кэш `is_premium`, сессий)              |
| **Ingestion Service**     | Приём, валидация и транскодирование аудиофайлов от авторов                     | Go                | S3 (временное хранение → перемещение в каталог), Kafka |
| **Catalog Service**       | Управление треками, альбомами, исполнителями, жанрами                          | Python            | PostgreSQL (источник истины), Elasticsearch (индекс поиска) |
| **Playback Service**      | Выдача URL аудиофайлов, логирование прослушиваний                              | Go                | S3 + CDN, Kafka (`PlaybackStarted`)            |
| **Users Service**         | Профили, плейлисты, импорт из VK / Яндекс.Музыки                               | Python            | PostgreSQL                                     |
| **Payments Service**      | Обработка подписок, интеграция с СБП / USDT                                    | Python            | PostgreSQL, Kafka (`PaymentCompleted`)         |
| **Ads Service**           | Выдача рекламных ассетов каждые 7 минут                                        | Go                | —                                              |
| **Recommendations Service** | Суточные батчи рекомендаций на основе истории                                | Python            | PostgreSQL (чтение прослушиваний), Kafka (подписка) |
| **Админка (UI + BFF)**    | Веб-интерфейс для авторов                                                      | TypeScript/React  | —                                              |

## Ключевые принципы
> Event-Driven: все важные события (TrackPublished, PaymentCompleted, PlaybackStarted) публикуются в Kafka.
> 
> Источник истины: PostgreSQL — единственное хранилище для транзакционных данных.
> 
> Read-оптимизация: Elasticsearch используется только для поиска, синхронизация — асинхронная.
> 
> Клиент решает о рекламе: флаг is_premium в JWT → плеер на стороне клиента вставляет рекламу или нет.
> 
> CDN: аудиофайлы доставляются через CDN поверх Yandex Object Storage (S3-совместимого).


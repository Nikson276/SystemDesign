---
title: "Вид выполнения (сценарии использования)"
status: draft
version: 0.1
author: "Михайлов Никита"
date: 2025-11-06
tags: [сценарии, runtime, диаграммы-последовательности, PlantUML]
---

# Вид выполнения (сценарии использования)

Ниже описаны ключевые сценарии взаимодействия компонентов в runtime. Каждый сценарий соответствует одной или нескольким [user story](./03_system_scope_and_context.md#user-story) и отражает поведение системы в production-условиях.

## Сценарий 0: Пользователь инициирует вход (Аутентификация)

**Связанные user story:** №12

> **Через Соцсети (Google, VK, Yandex)**
> 
> **Описание:** Пользователь нажимает кнопку "Войти через соцсеть" на фронтенде, система перенаправляет его на страницу авторизации выбранного провайдера (Google, VK, Yandex)
> 
> **Через Email + Пароль**
> 
> **Описание:** Пользователь вводит email и пароль, система проверяет учетные данные и возвращает JWT.


![Аутентификация](./_media/auth_secuence.png)

```plantuml
@startuml
title Единый поток аутентификации (Соцсети + Email/Пароль)

actor User as U
participant "Frontend" as FE
participant "Auth Service" as AS
participant "OAuth Provider\n(Google/VK/Yandex)" as OP
participant "Users Service" as US

== Вход через соцсеть (OIDC) ==
U -> FE: Нажимает "Войти через соцсеть"
FE -> AS: Запрос на авторизацию через выбранного провайдера
AS -> OP: Перенаправление (OIDC Authorization Request)
OP -> U: Форма авторизации
U -> OP: Ввод данных и подтверждение
OP -> FE: Authorization Code (редирект)
FE -> AS: Передача Authorization Code
AS -> OP: Обмен Code на Access Token + ID Token
OP -> AS: Токены + профиль (OIDC UserInfo)
AS -> US: Create/Update пользователя с данными профиля
US -> AS: Возвращает internal user_id

== Вход по Email/Пароль ==
U -> FE: Ввод email и пароль
FE -> AS: Запрос логина (email, password)
AS -> US: Запрос пользователя по email
US -> AS: Возвращает профиль и password_hash
AS -> AS: Проверка пароля (bcrypt/scrypt/argon2)
AS -> US: Получение is_premium, region, plan_type

== Общий этап генерации JWT ==
AS -> FE: Выдача JWT (user_id_internal, is_premium, plan_type, region)
note right of AS: JWT — единый формат для всех способов входа. В системе используется внутренний user_id.
@enduml
```

> Два способа аутентификации идут в параллели верхней части схемы.
> 
> Auth Service всегда работает через Users Service:
>   - для соцсетей — создаёт/обновляет профиль;
>   - для email — получает профиль и хэш пароля.

> JWT генерируется по единому стандарту:

```json
{
  "sub": "user_id_internal",
  "is_premium": false,
  "plan_type": "basic",
  "region": "RU",
  "exp": 1700000000
}
```

> Остальные микросервисы даже не знают, каким способом пользователь входил — смотрят только на claims в JWT.

## Сценарий 1: Пользователь слушает трек (free-аккаунт)

**Связанные user story**: №2, №6

**Описание**: Пользователь без премиум-подписки запрашивает воспроизведение трека. Система возвращает URL аудиофайла и рекламного ассета каждые 7 минут.

```plantuml
@startuml
actor Пользователь
participant "API Gateway" as gateway
participant "Playback Service" as playback
participant "Ads Service" as ads
participant "Object Storage (CDN)" as cdn

Пользователь -> gateway: GET /stream/{track_id} (JWT без is_premium)
gateway -> playback: Прокси-запрос
playback -> cdn: Получить pre-signed URL
cdn --> playback: URL аудио
playback --> gateway: Ответ с URL

gateway -> ads: GET /ad
ads --> gateway: Рекламный ассет (аудио/видео)
gateway --> Пользователь: Ответ: {audio_url, ad_url, interval=420s}

note right
  Клиентский плеер сначала проигрывает рекламу,
  затем — трек. Через 7 минут — снова реклама.
end note
@enduml
```
> Примечание: Решение о показе рекламы принимается на стороне клиента на основе отсутствия флага is_premium в JWT.

## Сценарий 2: Пользователь оформляет премиум-подписку

**Связанные user story:** №7, №8

**Описание:** Пользователь инициирует оплату через СБП. После успешного платежа его статус обновляется, и он получает доступ без рекламы.

```plantuml
@startuml
actor Пользователь
participant "API Gateway" as gateway
participant "Payments Service" as payments
participant "СБП" as sbp
participant "Users Service" as users
queue "Kafka" as kafka

Пользователь -> gateway: POST /subscribe {method: "SBP"}
gateway -> payments: Создать платёж
payments -> sbp: Инициировать платёж (Open API)
sbp --> payments: Подтверждение (payment_id)
payments -> kafka: Опубликовать PaymentCompleted {user_id, status=success}
payments --> gateway: Ответ: "Ожидайте подтверждения"
gateway --> Пользователь: Ссылка на СБП (deep link)

... Платёж подтверждён в СБП ...

sbp -> payments: Webhook: payment success
payments -> kafka: PaymentCompleted (финальный)
kafka -> users: Подписка на PaymentCompleted
users -> users: Обновить is_premium = true в БД
users -> gateway: (опционально) уведомление

note right
  При следующем запросе JWT будет содержать
  is_premium = true → клиент отключит рекламу.
end note
@enduml
```

## Сценарий 3: Автор загружает новый трек через админку

**Связанные user story:** №10, №11

**Описание:** Автор загружает аудиофайл и метаданные. Трек сохраняется, обрабатывается и публикуется в каталоге через событие.

```plantuml
@startuml
actor Автор
participant "Админка (BFF)" as admin
participant "Ingestion Service" as ingestion
participant "Object Storage" as s3
queue "Kafka" as kafka
participant "Catalog Service" as catalog
database "PostgreSQL" as pg
database "Elasticsearch" as es

Автор -> admin: Загрузить файл + метаданные
admin -> ingestion: POST /tracks (multipart)
ingestion -> s3: Сохранить аудио (временный ключ)
s3 --> ingestion: OK
ingestion -> ingestion: Валидация, транскодирование (если нужно)
ingestion -> pg: Сохранить черновик трека
pg --> ingestion: track_id
ingestion -> kafka: TrackPublished {track_id, status=published}
ingestion --> admin: OK
admin --> Автор: "Трек опубликован!"

kafka -> catalog: Подписка на TrackPublished
catalog -> pg: Загрузить метаданные трека
catalog -> es: Обновить поисковый индекс
@enduml
```

>  Трек становится доступен в поиске и каталоге после индексации в Elasticsearch.


## Сценарий 4: Система формирует суточные рекомендации

**Связанные user story:** №5

**Описание:** Раз в сутки запускается batch-процесс, который на основе истории прослушиваний генерирует персонализированные рекомендации.

```plantuml
@startuml
participant "Scheduler" as cron
participant "Recommendations Service" as rec
database "PostgreSQL" as pg
queue "Kafka" as kafka

cron -> rec: Запуск суточного джоба (02:00 MSK)
rec -> pg: SELECT user_id, track_id, timestamp FROM playback_events (последние 7 дней)
pg --> rec: Данные истории
rec -> rec: ML-модель (collaborative filtering / content-based)
rec -> pg: Сохранить рекомендации: {user_id, [track_ids]}
pg --> rec: OK

note right
  При следующем запросе /recommendations
  Catalog Service возвращает треки по ID.
end note
@enduml
```

> Важно: рекомендации не генерируются в реальном времени — это offline-батч.


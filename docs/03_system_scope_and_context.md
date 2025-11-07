---
title: "Границы системы и контекст"
status: draft
version: 0.1
author: "Михайлов Никита"
date: 2025-11-06
tags: [контекст, внешние-системы, C4, диаграмма]
---

# Границы системы и контекст

Платформа **ТУНЕЦ** — это единая система, предоставляющая музыкальный streaming-сервис с поддержкой прослушивания, поиска, рекомендаций, рекламы и премиум-подписок.

## Диаграмма контекста (C4 Level 1)

<details>
<summary>Диаграмма C1</summary>

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

Person(user, "Пользователь", "Слушает музыку, создаёт плейлисты, оплачивает подписку")
Person(artist, "Музыкант / Автор", "Загружает треки через админку")

System_Boundary(tunec, "ТУНЕЦ") {
  System(admin_ui, "Админка (UI + BFF)", "Веб-интерфейс для загрузки контента")
  System(api_gateway, "API Gateway", "Единая точка входа, аутентификация, маршрутизация")
}

System(ym, "Яндекс.Музыка", "Источник данных для импорта избранных треков")
System(vk, "VK", "Источник данных для импорта избранных треков")
System(sbp, "СБП", "Платёжная система")
System(usdt, "USDT (криптовалюта)", "Альтернативный способ оплаты")
System(yc, "Яндекс.Облако", "Инфраструктурный провайдер (Kafka, S3, PostgreSQL и т.д.)")

Rel(user, api_gateway, "Использует", "HTTPS")
Rel(artist, admin_ui, "Загружает треки", "HTTPS")
Rel(admin_ui, api_gateway, "Вызывает Ingestion API", "HTTPS")

Rel_L(api_gateway, ym, "Импорт по согласию", "OAuth + API")
Rel_R(api_gateway, vk, "Импорт по согласию", "OAuth + API")

Rel(api_gateway, sbp, "Инициирует платёж", "API")
Rel(api_gateway, usdt, "Принимает крипто", "Blockchain API")

Rel(tunec, yc, "Развёрнут в", "Инфраструктура")
@enduml
```

</details>

## Границы системы

> Внутри границ:
> 
> все микросервисы (Catalog, Playback, Users, Payments, Ads, Recommendations, Ingestion), API Gateway, админка (BFF + UI).
> 
> Вне границ:
> 
> платёжные системы, соцсети, облачный провайдер (как внешний сервис, хотя хостинг — внутри его экосистемы).

> ✅ Система не зависит от внешних CDN или аналитики — CDN интегрируется как часть Яндекс.Облака, аналитика — внутренняя.
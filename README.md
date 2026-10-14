# Classified — сервис объявлений

Classified - упрощённый аналог Avito. Пользователь может найти объявление, посмотреть карточку товара и фотографии, а после авторизации — опубликовать объявление, получить персональную ленту и посмотреть свои публикации. Объявление активно 30 дней.

## Архитектура

- **API Gateway** — входная точка для HTTP-запросов.
- **Auth & IAM** — авторизация; для защищённых ручек используется JWT.
- **Search Service + Elasticsearch** — главная страница, поиск по названию/описанию и фильтрация объявлений.
- **Product Service + PostgreSQL** — создание, чтение карточек и список объявлений пользователя. PostgreSQL — источник истины; данные реплицируются.
- **S3 + CDN** — фотографии: загрузка напрямую в S3 по presigned URL, просмотр через CDN.
- **UserEvents Service → Kafka → ClickHouse** — просмотры, клики и другие действия пользователей для аналитики.
- **PostgreSQL WAL → Debezium → Kafka → Elasticsearch / ClickHouse** — асинхронное обновление поискового индекса и аналитических данных.
- **FeedWorker → Redis → Feed Service** — FeedWorker периодически строит рекомендации на основе ClickHouse и сохраняет ID объявлений в Redis. Feed Service получает эти ID и актуальные данные объявлений из Elasticsearch.

## Основные API

| Метод | Маршрут | Назначение | Авторизация |
|---|---|---|---|
| `GET` | `/api/v1/products` | Главная страница и поиск | Нет |
| `GET` | `/api/v1/product/{id}` | Карточка объявления | Нет |
| `GET` | `/api/v1/feed` | Персональная лента | JWT |
| `GET` | `/api/v1/listings` | Свои объявления | JWT |
| `POST` | `/api/v1/product` | Создать объявление | JWT |
| `GET` | `/api/v1/products/upload-url` | Получить presigned URL для фото | JWT |

После получения `uploadUrl` клиент делает `PUT` фотографии **напрямую в S3**, минуя API Gateway. Затем при создании объявления передаёт ключи загруженных изображений.


## Rate Limiter и Circuit Breaker

**Rate Limiter:** на **API Gateway**, до вызова внутренних сервисов. Для публичных запросов — лимит по IP, для авторизованных — по `user_id`. Отдельные лимиты можно назначить на поиск, публикацию и выдачу `upload-url`. 

**Circuit Breaker:** на **исходящих вызовах сервисов к зависимостям**:

- `Search Service → Elasticsearch`;
- `Feed Service → Redis / Elasticsearch`;
- `FeedWorker → ClickHouse / Redis`;
- `Product Service → Redis / PostgreSQL` 

При открытом Circuit Breaker запрос быстро завершается ошибкой или использует **заранее предусмотренный fallback**: например, Feed Service может вернуть обычную выдачу вместо персональной.

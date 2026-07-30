# StoreForge — Architecture Design

## 1. Назначение документа

Этот документ фиксирует начальную микросервисную архитектуру учебного интернет-магазина **StoreForge**.

Главная цель первой версии — реализовать основной пользовательский сценарий:

```text
Регистрация
→ Просмотр каталога
→ Добавление товара в корзину
→ Создание заказа
→ Резервирование товара
→ Оплата
→ Подтверждение заказа
→ Уведомление пользователя
```

На первом этапе не требуется реализовывать все возможные функции интернет-магазина. Сначала необходимо построить минимальный рабочий контур и только после этого расширять систему.

---

## 2. Основные архитектурные правила

1. Каждый микросервис хранится в отдельном Git-репозитории.
2. Каждый сервис отвечает только за свою предметную область.
3. Каждый сервис владеет своими данными.
4. Сервис не обращается напрямую к таблицам другого сервиса.
5. Синхронные запросы выполняются через HTTP API.
6. Асинхронные бизнес-процессы выполняются через RabbitMQ.
7. Внешний клиент обращается к backend через API Gateway.
8. Каждый сервис независимо собирается в Docker-образ.
9. События RabbitMQ должны иметь версию контракта.
10. Обработчики сообщений должны быть идемпотентными.

---

## 3. Общая схема системы

```text
                         Browser / Frontend
                                 |
                                 v
                            API Gateway
                                 |
       +-------------------------+---------------------------+
       |             |              |             |          |
       v             v              v             v          v
 Auth Service   Catalog Service  Cart Service  Order Service Review Service
                                                    |
                                                    v
                                                RabbitMQ
                                                    |
                         +--------------------------+---------------------+
                         |                          |                     |
                         v                          v                     v
                 Inventory Service          Payment Service     Notification Service


Infrastructure:
- PostgreSQL
- Redis
- RabbitMQ
- MinIO
- OpenTelemetry Collector
- Prometheus / Grafana
```

`Review Service`, MinIO и расширенный observability-контур не являются обязательными для первой версии.

---

# 4. Сервисы

## 4.1 Frontend

### Назначение

Пользовательский интерфейс интернет-магазина.

Frontend отвечает за:

- регистрацию и авторизацию пользователя;
- отображение каталога;
- отображение карточки товара;
- работу с корзиной;
- оформление заказа;
- отображение статуса заказа;
- отображение истории заказов.

### Хранилище

Собственной базы данных нет.

Локально frontend может хранить только технические данные:

- access token;
- состояние интерфейса;
- временные пользовательские настройки.

### HTTP API

Frontend обращается только к API Gateway:

```text
/api/v1/auth/*
/api/v1/products/*
/api/v1/cart/*
/api/v1/orders/*
/api/v1/payments/*
```

### Публикуемые события

Нет.

### Получаемые события

Нет. События RabbitMQ недоступны браузеру напрямую.

---

## 4.2 API Gateway

### Назначение

Единая точка входа во внутренние микросервисы.

API Gateway отвечает за:

- маршрутизацию HTTP-запросов;
- проверку базового формата JWT;
- передачу запросов нужному сервису;
- CORS;
- rate limiting в будущем;
- формирование общего внешнего API.

Gateway не должен содержать основную бизнес-логику интернет-магазина.

### База данных

Нет.

Для rate limiting в будущем может использовать Redis.

### HTTP endpoints

```text
/api/v1/auth/*       -> Auth Service
/api/v1/users/*      -> Auth Service
/api/v1/products/*   -> Catalog Service
/api/v1/categories/* -> Catalog Service
/api/v1/cart/*       -> Cart Service
/api/v1/orders/*     -> Order Service
/api/v1/payments/*   -> Payment Service
/api/v1/reviews/*    -> Review Service
```

### Публикуемые события

Нет.

### Получаемые события

Нет.

---

## 4.3 Auth Service

### Назначение

Управление пользователями и аутентификацией.

Auth Service отвечает за:

- регистрацию;
- авторизацию;
- password hashing;
- access JWT;
- refresh token;
- роли пользователей;
- блокировку пользователя;
- получение текущего пользователя.

### База данных

```text
auth_db
```

Основные таблицы:

```text
users
roles
user_roles
refresh_tokens
outbox_events
processed_events
```

### Владелец данных

Auth Service является единственным владельцем:

- email пользователя;
- хеша пароля;
- статуса пользователя;
- ролей;
- refresh-токенов.

Другие сервисы хранят только `user_id` и необходимые локальные снимки данных.

### HTTP endpoints

```http
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
GET  /api/v1/users/me
PATCH /api/v1/users/me
GET  /internal/v1/users/{user_id}
```

### Публикуемые события

```text
user.registered.v1
user.updated.v1
user.blocked.v1
user.unblocked.v1
```

### Получаемые события

В первой версии отсутствуют.

---

## 4.4 Catalog Service

### Назначение

Управление каталогом интернет-магазина.

Catalog Service отвечает за:

- товары;
- категории;
- характеристики;
- базовые цены;
- изображения товаров;
- поиск;
- фильтрацию;
- пагинацию;
- административный CRUD.

Catalog Service не отвечает за фактические складские остатки.

### База данных

```text
catalog_db
```

Основные таблицы:

```text
products
categories
product_categories
product_images
product_attributes
outbox_events
processed_events
```

Redis используется для кеширования:

```text
catalog:product:{product_id}
catalog:category:{category_id}
catalog:search:{query_hash}
```

Файлы изображений в будущем хранятся в MinIO/S3. В PostgreSQL хранится только object key или URL.

### Владелец данных

Catalog Service владеет:

- названием товара;
- описанием;
- категориями;
- характеристиками;
- базовой ценой;
- ссылками на изображения;
- признаком активности товара.

### HTTP endpoints

```http
GET    /api/v1/products
GET    /api/v1/products/{product_id}
GET    /api/v1/categories
GET    /api/v1/categories/{category_id}/products
POST   /api/v1/admin/products
PATCH  /api/v1/admin/products/{product_id}
DELETE /api/v1/admin/products/{product_id}
POST   /api/v1/admin/categories
PATCH  /api/v1/admin/categories/{category_id}
GET    /internal/v1/products/{product_id}
POST   /internal/v1/products/batch
```

### Публикуемые события

```text
product.created.v1
product.updated.v1
product.deleted.v1
product.price_changed.v1
category.created.v1
category.updated.v1
```

### Получаемые события

В первой версии отсутствуют.

---

## 4.5 Cart Service

### Назначение

Управление текущей корзиной пользователя.

Cart Service отвечает за:

- добавление товара;
- изменение количества;
- удаление товара;
- получение корзины;
- очистку корзины;
- временное хранение состава корзины.

Корзина не является заказом и не фиксирует окончательную цену товара.

### База данных

Основное хранилище:

```text
Redis
```

Ключ:

```text
cart:{user_id}
```

Пример содержимого:

```json
{
  "items": [
    {
      "product_id": "uuid",
      "quantity": 2
    }
  ]
}
```

PostgreSQL в первой версии не требуется.

### Владелец данных

Cart Service владеет:

- текущим составом корзины;
- количеством товаров в корзине;
- временем последнего изменения корзины.

### HTTP endpoints

```http
GET    /api/v1/cart
POST   /api/v1/cart/items
PATCH  /api/v1/cart/items/{product_id}
DELETE /api/v1/cart/items/{product_id}
DELETE /api/v1/cart
GET    /internal/v1/carts/{user_id}
DELETE /internal/v1/carts/{user_id}
```

### Публикуемые события

Для первой версии не обязательны.

В дальнейшем:

```text
cart.updated.v1
cart.abandoned.v1
```

### Получаемые события

```text
order.created.v1
```

После подтверждённого создания заказа Cart Service может очистить корзину пользователя.

---

## 4.6 Order Service

### Назначение

Управление заказами и координация процесса оформления заказа.

Order Service является оркестратором checkout-процесса.

Он отвечает за:

- создание заказа;
- состав заказа;
- фиксацию цены на момент покупки;
- расчёт итоговой суммы;
- адрес доставки;
- статусы заказа;
- историю изменения статусов;
- отмену заказа;
- запуск резервирования;
- запуск оплаты;
- обработку результатов склада и оплаты.

### База данных

```text
order_db
```

Основные таблицы:

```text
orders
order_items
order_status_history
outbox_events
processed_events
idempotency_keys
```

### Владелец данных

Order Service владеет:

- заказом;
- позициями заказа;
- итоговой стоимостью;
- статусом заказа;
- историей статусов;
- снимками товара на момент заказа;
- адресом доставки.

В `order_items` сохраняются снимки:

```text
product_id
product_name
unit_price
quantity
line_total
```

Order Service не должен при просмотре старого заказа получать актуальное название или цену из Catalog Service.

### HTTP endpoints

```http
POST /api/v1/orders
GET  /api/v1/orders
GET  /api/v1/orders/{order_id}
POST /api/v1/orders/{order_id}/cancel
GET  /internal/v1/orders/{order_id}
```

При создании заказа рекомендуется принимать заголовок:

```http
Idempotency-Key: <uuid>
```

### Публикуемые команды и события

Команды:

```text
inventory.reserve.requested.v1
inventory.release.requested.v1
inventory.commit.requested.v1
payment.create.requested.v1
payment.cancel.requested.v1
```

События:

```text
order.created.v1
order.awaiting_payment.v1
order.paid.v1
order.cancelled.v1
order.failed.v1
order.completed.v1
```

### Получаемые события

```text
inventory.reserved.v1
inventory.reservation_failed.v1
inventory.released.v1
inventory.committed.v1
payment.created.v1
payment.succeeded.v1
payment.failed.v1
payment.cancelled.v1
```

---

## 4.7 Inventory Service

### Назначение

Управление складскими остатками и резервами.

Inventory Service отвечает за:

- доступное количество товара;
- резервирование;
- освобождение резерва;
- окончательное списание;
- поступление товара;
- историю складских операций;
- защиту от отрицательных остатков.

### База данных

```text
inventory_db
```

Основные таблицы:

```text
inventory_items
inventory_reservations
inventory_reservation_items
inventory_movements
outbox_events
processed_events
```

### Владелец данных

Inventory Service является единственным владельцем:

- доступного количества товара;
- зарезервированного количества;
- складских движений;
- резервов заказа.

Catalog Service не должен хранить фактический остаток как источник истины.

### HTTP endpoints

Публичное API минимально:

```http
GET   /api/v1/products/{product_id}/availability
GET   /api/v1/admin/inventory
POST  /api/v1/admin/inventory/receipts
PATCH /api/v1/admin/inventory/{product_id}
```

Внутреннее API:

```http
GET /internal/v1/inventory/{product_id}
```

Основные бизнес-операции выполняются через RabbitMQ.

### Публикуемые события

```text
inventory.reserved.v1
inventory.reservation_failed.v1
inventory.released.v1
inventory.committed.v1
inventory.stock_changed.v1
```

### Получаемые команды и события

```text
inventory.reserve.requested.v1
inventory.release.requested.v1
inventory.commit.requested.v1
order.cancelled.v1
payment.failed.v1
```

---

## 4.8 Payment Service

### Назначение

Управление платежами.

В первой версии Payment Service имитирует работу внешнего платёжного провайдера.

Сервис отвечает за:

- создание платежа;
- статус платежа;
- привязку платежа к заказу;
- успешную оплату;
- ошибку оплаты;
- отмену;
- возврат в будущем;
- идемпотентность платёжных операций.

### База данных

```text
payment_db
```

Основные таблицы:

```text
payments
payment_attempts
payment_webhooks
refunds
outbox_events
processed_events
idempotency_keys
```

### Владелец данных

Payment Service владеет:

- платёжной операцией;
- идентификатором внешнего провайдера;
- статусом платежа;
- попытками оплаты;
- возвратами.

Order Service хранит только `payment_id` и бизнес-статус заказа.

### HTTP endpoints

```http
GET  /api/v1/payments/{payment_id}
POST /api/v1/payments/{payment_id}/mock-success
POST /api/v1/payments/{payment_id}/mock-fail
POST /api/v1/payments/{payment_id}/cancel
POST /api/v1/payments/webhooks/provider
GET  /internal/v1/payments/{payment_id}
```

### Публикуемые события

```text
payment.created.v1
payment.succeeded.v1
payment.failed.v1
payment.cancelled.v1
payment.refunded.v1
```

### Получаемые команды

```text
payment.create.requested.v1
payment.cancel.requested.v1
```

---

## 4.9 Notification Service

### Назначение

Асинхронная отправка пользовательских уведомлений.

Notification Service отвечает за:

- email-уведомления;
- шаблоны сообщений;
- повторные попытки доставки;
- журнал отправки;
- Telegram-уведомления в будущем.

### База данных

```text
notification_db
```

Основные таблицы:

```text
notifications
notification_templates
notification_attempts
processed_events
```

### Владелец данных

Notification Service владеет:

- уведомлениями;
- результатами доставки;
- количеством попыток;
- используемыми шаблонами.

### HTTP endpoints

Публичное API в первой версии не требуется.

Технические endpoints:

```http
GET /health/live
GET /health/ready
GET /internal/v1/notifications/{notification_id}
```

### Публикуемые события

В будущем:

```text
notification.sent.v1
notification.failed.v1
```

### Получаемые события

```text
user.registered.v1
order.created.v1
order.awaiting_payment.v1
order.paid.v1
order.cancelled.v1
order.completed.v1
payment.failed.v1
inventory.reservation_failed.v1
```

---

## 4.10 Review Service

### Статус

Не входит в первую итерацию разработки.

### Назначение

Управление отзывами и рейтингами товаров.

### База данных

```text
review_db
```

Основные таблицы:

```text
reviews
product_rating_summary
completed_order_items_projection
outbox_events
processed_events
```

### Владелец данных

Review Service владеет:

- отзывами;
- оценками;
- агрегированным рейтингом;
- локальной проекцией завершённых покупок.

### HTTP endpoints

```http
GET    /api/v1/products/{product_id}/reviews
POST   /api/v1/products/{product_id}/reviews
PATCH  /api/v1/reviews/{review_id}
DELETE /api/v1/reviews/{review_id}
```

### Публикуемые события

```text
review.created.v1
review.updated.v1
review.deleted.v1
product.rating_changed.v1
```

### Получаемые события

```text
order.completed.v1
```

---

# 5. Владение данными

| Данные | Владелец |
|---|---|
| Пользователи и пароли | Auth Service |
| Роли и refresh-токены | Auth Service |
| Товары и категории | Catalog Service |
| Цена товара | Catalog Service |
| Изображения товара | Catalog Service + MinIO |
| Текущая корзина | Cart Service |
| Заказы и позиции заказа | Order Service |
| Снимок цены в заказе | Order Service |
| Доступный складской остаток | Inventory Service |
| Резервы товара | Inventory Service |
| Платежи и попытки оплаты | Payment Service |
| Уведомления | Notification Service |
| Отзывы и рейтинг | Review Service |

Главное правило:

```text
Ни один сервис не подключается к базе другого сервиса.
```

---

# 6. HTTP-взаимодействие

HTTP используется, когда ответ требуется сразу.

Примеры:

```text
Frontend -> API Gateway
API Gateway -> Auth Service
API Gateway -> Catalog Service
API Gateway -> Cart Service
API Gateway -> Order Service
Cart Service -> Catalog Service
Order Service -> Cart Service
Order Service -> Catalog Service
```

Необходимо избегать длинных синхронных цепочек.

Плохой пример:

```text
Order -> Cart -> Catalog -> Inventory -> Auth
```

Если один сервис в цепочке недоступен, падает весь запрос.

Для внутренних API используется префикс:

```text
/internal/v1
```

Для публичного API:

```text
/api/v1
```

---

# 7. RabbitMQ

## 7.1 Exchanges

Предлагаемые exchanges:

```text
storeforge.events
storeforge.commands
storeforge.dlx
```

Тип exchange:

```text
topic
```

## 7.2 Очереди

```text
order.inventory-results
order.payment-results
inventory.commands
payment.commands
notification.events
cart.events
review.events
```

У каждого consumer должна быть собственная очередь.

Нельзя использовать одну общую очередь для всех сервисов, потому что одно сообщение получит только один consumer.

## 7.3 Routing keys

Примеры:

```text
order.created.v1
order.cancelled.v1
inventory.reserve.requested.v1
inventory.reserved.v1
payment.create.requested.v1
payment.succeeded.v1
```

## 7.4 Общий формат сообщения

```json
{
  "event_id": "uuid",
  "event_type": "order.created.v1",
  "occurred_at": "2026-07-30T00:00:00Z",
  "producer": "order-service",
  "correlation_id": "uuid",
  "causation_id": "uuid",
  "trace_id": "otel-trace-id",
  "payload": {}
}
```

Обязательные поля:

```text
event_id
event_type
occurred_at
producer
correlation_id
payload
```

---

# 8. Основные события

## 8.1 user.registered.v1

```json
{
  "user_id": "uuid",
  "email": "user@example.com"
}
```

Потребители:

```text
Notification Service
```

## 8.2 order.created.v1

```json
{
  "order_id": "uuid",
  "user_id": "uuid",
  "total_amount": "3990.00",
  "currency": "RUB"
}
```

Потребители:

```text
Cart Service
Notification Service
Analytics в будущем
```

## 8.3 inventory.reserve.requested.v1

```json
{
  "order_id": "uuid",
  "items": [
    {
      "product_id": "uuid",
      "quantity": 2
    }
  ]
}
```

Потребитель:

```text
Inventory Service
```

## 8.4 inventory.reserved.v1

```json
{
  "order_id": "uuid",
  "reservation_id": "uuid",
  "expires_at": "2026-07-30T01:00:00Z"
}
```

Потребитель:

```text
Order Service
```

## 8.5 inventory.reservation_failed.v1

```json
{
  "order_id": "uuid",
  "reason": "insufficient_stock",
  "failed_items": [
    {
      "product_id": "uuid",
      "requested": 2,
      "available": 1
    }
  ]
}
```

Потребители:

```text
Order Service
Notification Service
```

## 8.6 payment.create.requested.v1

```json
{
  "order_id": "uuid",
  "user_id": "uuid",
  "amount": "3990.00",
  "currency": "RUB"
}
```

Потребитель:

```text
Payment Service
```

## 8.7 payment.succeeded.v1

```json
{
  "payment_id": "uuid",
  "order_id": "uuid",
  "amount": "3990.00",
  "currency": "RUB",
  "paid_at": "2026-07-30T00:10:00Z"
}
```

Потребители:

```text
Order Service
Notification Service
```

## 8.8 payment.failed.v1

```json
{
  "payment_id": "uuid",
  "order_id": "uuid",
  "reason": "payment_declined"
}
```

Потребители:

```text
Order Service
Inventory Service
Notification Service
```

---

# 9. Процесс оформления заказа

## 9.1 Успешный сценарий

```text
1. Пользователь вызывает POST /api/v1/orders.

2. Order Service получает корзину из Cart Service.

3. Order Service получает актуальные данные товаров из Catalog Service.

4. Order Service рассчитывает итоговую сумму.

5. Order Service создаёт заказ в статусе CREATED.

6. В одной транзакции создаются:
   - заказ;
   - позиции заказа;
   - запись в outbox_events.

7. Order Service публикует inventory.reserve.requested.v1.

8. Inventory Service проверяет остатки и создаёт резерв.

9. Inventory Service публикует inventory.reserved.v1.

10. Order Service переводит заказ в AWAITING_PAYMENT.

11. Order Service публикует payment.create.requested.v1.

12. Payment Service создаёт платёж.

13. Payment Service публикует payment.created.v1.

14. Пользователь выполняет mock-оплату.

15. Payment Service публикует payment.succeeded.v1.

16. Order Service переводит заказ в PAID.

17. Order Service публикует inventory.commit.requested.v1.

18. Inventory Service окончательно списывает резерв.

19. Inventory Service публикует inventory.committed.v1.

20. Notification Service отправляет уведомление.

21. Cart Service очищает корзину.
```

## 9.2 Недостаточно товара

```text
Order Service
    -> inventory.reserve.requested.v1

Inventory Service
    -> inventory.reservation_failed.v1

Order Service
    -> статус FAILED

Notification Service
    -> уведомление пользователю
```

## 9.3 Ошибка оплаты

```text
Payment Service
    -> payment.failed.v1

Order Service
    -> статус PAYMENT_FAILED
    -> inventory.release.requested.v1

Inventory Service
    -> освобождает резерв
    -> inventory.released.v1

Order Service
    -> статус CANCELLED
```

## 9.4 Отмена пользователем

Если заказ ещё не оплачен:

```text
User
    -> POST /orders/{order_id}/cancel

Order Service
    -> inventory.release.requested.v1
    -> payment.cancel.requested.v1
    -> статус CANCELLED
```

Если заказ уже оплачен, в первой версии отмену можно запретить. Возвраты добавить позднее.

---

# 10. Статусы заказа

```text
CREATED
RESERVING
AWAITING_PAYMENT
PAID
PROCESSING
COMPLETED
PAYMENT_FAILED
CANCELLED
FAILED
```

## Допустимые переходы

```text
CREATED
  -> RESERVING

RESERVING
  -> AWAITING_PAYMENT
  -> FAILED

AWAITING_PAYMENT
  -> PAID
  -> PAYMENT_FAILED
  -> CANCELLED

PAYMENT_FAILED
  -> CANCELLED

PAID
  -> PROCESSING

PROCESSING
  -> COMPLETED

Любой неоплаченный статус
  -> CANCELLED
```

Статус нельзя изменять произвольно. Переходы должны проверяться бизнес-логикой Order Service.

---

# 11. Статусы резервирования

```text
PENDING
RESERVED
COMMITTED
RELEASED
EXPIRED
FAILED
```

---

# 12. Статусы платежа

```text
PENDING
PROCESSING
SUCCEEDED
FAILED
CANCELLED
REFUNDED
```

---

# 13. Надёжность сообщений

## 13.1 Transactional Outbox

Для критических сервисов используется таблица:

```text
outbox_events
```

Изменение бизнес-данных и создание события выполняются в одной SQL-транзакции.

Outbox необходим для:

```text
Order Service
Inventory Service
Payment Service
Auth Service
```

## 13.2 Идемпотентность consumers

Каждый consumer хранит обработанные сообщения:

```text
processed_events
```

Минимальные поля:

```text
event_id
consumer_name
processed_at
```

Уникальный индекс:

```text
(event_id, consumer_name)
```

## 13.3 Retry и DLQ

Для каждой очереди предусматриваются:

```text
основная очередь
retry-очередь
dead-letter queue
```

Пример:

```text
payment.commands
payment.commands.retry
payment.commands.dlq
```

Сообщение попадает в DLQ после исчерпания числа повторных попыток.

---

# 14. Структура Polyrepo

```text
storeforge-frontend
storeforge-api-gateway
storeforge-auth-service
storeforge-catalog-service
storeforge-cart-service
storeforge-order-service
storeforge-inventory-service
storeforge-payment-service
storeforge-notification-service
storeforge-infrastructure
```

Позднее:

```text
storeforge-review-service
storeforge-contracts
```

---

# 15. Структура репозитория микросервиса

Пример для Order Service:

```text
storeforge-order-service/
├── src/
│   └── order_service/
│       ├── main.py
│       ├── api/
│       │   ├── dependencies.py
│       │   └── v1/
│       ├── application/
│       │   ├── commands/
│       │   ├── queries/
│       │   └── services/
│       ├── domain/
│       │   ├── entities/
│       │   ├── enums/
│       │   └── exceptions/
│       ├── infrastructure/
│       │   ├── database/
│       │   ├── repositories/
│       │   ├── messaging/
│       │   └── http_clients/
│       ├── consumers/
│       ├── schemas/
│       ├── observability/
│       └── config.py
├── migrations/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── api/
├── Dockerfile
├── compose.yml
├── alembic.ini
├── pyproject.toml
├── .env.example
├── README.md
└── .github/workflows/ci.yml
```

Для первой версии допустима более простая структура:

```text
src/
├── api/
├── models/
├── schemas/
├── repositories/
├── services/
├── consumers/
├── messaging/
├── database/
└── config.py
```

---

# 16. Структура infrastructure-репозитория

```text
storeforge-infrastructure/
├── compose.yml
├── .env.example
├── postgres/
│   └── init-databases.sql
├── rabbitmq/
│   ├── definitions.json
│   └── rabbitmq.conf
├── redis/
├── minio/
├── nginx/
│   └── nginx.conf
├── monitoring/
│   ├── prometheus/
│   ├── grafana/
│   └── otel-collector/
├── helm/
├── kubernetes/
└── README.md
```

В первой версии через Docker Compose запускаются:

```text
PostgreSQL
Redis
RabbitMQ
RabbitMQ Management UI
Mailpit
API Gateway
Frontend
разрабатываемые микросервисы
```

---

# 17. Базы данных в локальной среде

На первом этапе можно использовать один PostgreSQL-контейнер, но отдельную базу для каждого сервиса:

```text
storeforge_auth
storeforge_catalog
storeforge_orders
storeforge_inventory
storeforge_payments
storeforge_notifications
```

Каждый сервис получает отдельного пользователя PostgreSQL и доступ только к собственной базе.

Пример:

```text
auth_service_user      -> storeforge_auth
catalog_service_user   -> storeforge_catalog
order_service_user     -> storeforge_orders
inventory_service_user -> storeforge_inventory
payment_service_user   -> storeforge_payments
```

---

# 18. Что реализовывать сначала

Не нужно начинать со всех сервисов одновременно.

## Первая итерация

```text
1. storeforge-infrastructure
2. storeforge-auth-service
3. storeforge-catalog-service
4. storeforge-frontend
5. storeforge-api-gateway
```

Результат:

```text
Пользователь регистрируется
→ авторизуется
→ открывает каталог
→ смотрит товар
```

## Вторая итерация

```text
6. storeforge-cart-service
7. storeforge-order-service
```

Результат:

```text
Пользователь добавляет товары в корзину
→ создаёт заказ
```

На этом этапе заказ можно создавать без RabbitMQ и без оплаты, чтобы сначала проверить HTTP API и модель данных.

## Третья итерация

```text
8. storeforge-inventory-service
9. RabbitMQ-интеграция Order <-> Inventory
```

Результат:

```text
Заказ
→ резервирование товара
→ успешный резерв или отказ
```

## Четвёртая итерация

```text
10. storeforge-payment-service
11. Saga оформления заказа
```

Результат:

```text
Заказ
→ резерв
→ оплата
→ подтверждение или компенсация
```

## Пятая итерация

```text
12. storeforge-notification-service
13. Transactional Outbox
14. Retry и DLQ
15. Idempotent consumers
```

## Шестая итерация

```text
16. OpenTelemetry
17. Метрики
18. CI/CD
19. Kubernetes
20. Review Service
```

---

# 19. Минимальная первая версия

Первая полноценная версия StoreForge считается готовой, если работает следующий сценарий:

```text
1. Пользователь регистрируется.
2. Пользователь авторизуется.
3. Администратор создаёт товар.
4. Администратор задаёт складской остаток.
5. Пользователь добавляет товар в корзину.
6. Пользователь создаёт заказ.
7. Inventory Service резервирует товар.
8. Payment Service создаёт платёж.
9. Пользователь подтверждает mock-оплату.
10. Заказ получает статус PAID.
11. Остаток товара списывается.
12. Пользователь получает email через Mailpit.
```

---

# 20. Что не входит в первую версию

На старте не реализуются:

- Kubernetes;
- Kafka;
- service mesh;
- GraphQL;
- полноценная интеграция с банком;
- сложные возвраты;
- бонусная система;
- промокоды;
- рекомендации;
- полнотекстовый поиск;
- несколько складов;
- доставка отдельным сервисом;
- Review Service;
- отдельный PostgreSQL-кластер для каждого сервиса.

Эти компоненты добавляются после реализации основного checkout-процесса.

---

# 21. Итоговое решение

Для первой версии используются следующие сервисы:

```text
Frontend
API Gateway
Auth Service
Catalog Service
Cart Service
Order Service
Inventory Service
Payment Service
Notification Service
```

Главный оркестратор бизнес-процесса:

```text
Order Service
```

Основной механизм асинхронного взаимодействия:

```text
RabbitMQ
```

Главный принцип владения данными:

```text
Каждый сервис владеет собственной базой и не читает чужие таблицы.
```

Рекомендуемый старт разработки:

```text
Infrastructure
→ Auth
→ Catalog
→ Frontend
→ Gateway
→ Cart
→ Order
→ Inventory
→ Payment
→ Notifications
```

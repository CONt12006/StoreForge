# StoreForge
FastAPI интернет-магазин с PostgreSQL, Redis, RabbitMQ и Docker.

storeforge-auth-service: https://github.com/CONt12006/storeforge-auth-service  
storeforge-catalog-service: https://github.com/CONt12006/storeforge-catalog-service

# Предлагаемая архитектура

```text
                         Клиент / Frontend
                                |
                                v
                         API Gateway
                                |
          ┌─────────────┬───────┼────────┬─────────────┐
          v             v       v        v             v
       Auth          Catalog   Cart    Orders       Reviews
      Service        Service  Service  Service       Service
                                          |
                                          v
                                     RabbitMQ
                                          |
                    ┌─────────────────────┼────────────────────┐
                    v                     v                    v
              Inventory Service    Payment Service    Notification Worker
```

Инфраструктура:

```text
PostgreSQL
Redis
RabbitMQ
Docker Compose
API Gateway
OpenTelemetry Collector
Prometheus / Grafana
```

# Какие сервисы делать

## 1. Auth Service

Отвечает за:

* регистрацию;
* авторизацию;
* JWT;
* refresh-токены;
* роли `customer`, `admin`;
* информацию о пользователе.

Хранилище:

```text
PostgreSQL: auth_db
Redis: refresh-токены и блокировка токенов
```

Основные методы:

```http
POST /auth/register
POST /auth/login
POST /auth/refresh
GET  /users/me
```

---

## 2. Catalog Service

Отвечает за:

* товары;
* категории;
* цены;
* изображения;
* характеристики;
* поиск и фильтрацию;
* административное управление товарами.

Хранилище:

```text
PostgreSQL: catalog_db
Redis: кеш популярных товаров и запросов
```

Основные методы:

```http
GET  /products
GET  /products/{id}
POST /admin/products
PATCH /admin/products/{id}
```

Каталог не должен хранить остатки товара. Остатки принадлежат Inventory Service.

---

## 3. Cart Service

Отвечает за корзину пользователя:

* добавление товара;
* изменение количества;
* удаление товара;
* очистку корзины.

Хранилище:

```text
Redis
```

Методы:

```http
GET    /cart
POST   /cart/items
PATCH  /cart/items/{product_id}
DELETE /cart/items/{product_id}
DELETE /cart
```

Для первой версии отдельная PostgreSQL-база ему не нужна.

---

## 4. Order Service

Главный бизнес-сервис системы.

Отвечает за:

* создание заказа;
* позиции заказа;
* адрес доставки;
* итоговую сумму;
* статусы;
* отмену заказа;
* координацию склада и оплаты.

Хранилище:

```text
PostgreSQL: order_db
```

Методы:

```http
POST /orders
GET  /orders
GET  /orders/{id}
POST /orders/{id}/cancel
```

Пример статусов:

```text
created
reserving
awaiting_payment
paid
processing
completed
cancelled
failed
```

Именно Order Service лучше сделать оркестратором процесса оформления заказа.

---

## 5. Inventory Service

Отвечает за:

* количество товара;
* резерв;
* освобождение резерва;
* окончательное списание;
* историю изменения остатков.

Хранилище:

```text
PostgreSQL: inventory_db
```

Он в основном работает через RabbitMQ.

Получает команды:

```text
inventory.reserve
inventory.release
inventory.commit
```

Отправляет события:

```text
inventory.reserved
inventory.reservation_failed
inventory.released
inventory.committed
```

---

## 6. Payment Service

Отвечает за:

* создание платежей;
* состояние оплаты;
* обработку успешной оплаты;
* обработку ошибки;
* возврат;
* защиту от повторных платежей.

Хранилище:

```text
PostgreSQL: payment_db
```

Для учебного проекта сначала сделай имитацию платёжной системы:

```http
POST /payments/{id}/success
POST /payments/{id}/fail
```

События:

```text
payment.created
payment.succeeded
payment.failed
payment.refunded
```

---

## 7. Notification Worker

Это может быть не полноценный публичный сервис, а worker.

Он слушает события:

```text
user.registered
order.created
payment.succeeded
payment.failed
order.cancelled
```

И отправляет:

* email;
* сообщения в Telegram;
* уведомления в лог для первой версии.

Отдельный HTTP API ему почти не нужен.

---

## 8. Review Service — позднее

Не начинай с него. Добавь после того, как основной процесс заказа полностью заработает.

Он отвечает за:

* отзывы;
* оценки;
* средний рейтинг;
* проверку, что пользователь действительно покупал товар.

# Как сервисы взаимодействуют

Используй два типа связи.

## HTTP — когда ответ нужен сразу

Например:

```text
Cart Service -> Catalog Service
```

Чтобы получить название и цену товаров.

```text
Order Service -> Catalog Service
```

Чтобы перед созданием заказа проверить текущую цену и существование товара.

Не строй слишком длинные HTTP-цепочки:

```text
Order -> Cart -> Catalog -> Inventory -> Auth
```

Это создаст каскадные падения.

## RabbitMQ — для бизнес-процессов

Используй его для оформления заказа:

```text
Order Service
    |
    | inventory.reserve
    v
Inventory Service
    |
    | inventory.reserved
    v
Order Service
    |
    | payment.create
    v
Payment Service
```

Так сервисы будут связаны слабее.

# Как будет оформляться заказ

Общий процесс:

```text
1. Пользователь отправляет POST /orders
2. Order Service получает содержимое корзины
3. Проверяет товары и цены через Catalog Service
4. Создаёт заказ со статусом reserving
5. Отправляет команду inventory.reserve
6. Inventory Service резервирует товары
7. Отправляет inventory.reserved
8. Order Service переводит заказ в awaiting_payment
9. Отправляет payment.create
10. Payment Service создаёт платёж
11. Пользователь подтверждает оплату
12. Payment Service отправляет payment.succeeded
13. Order Service переводит заказ в paid
14. Inventory Service окончательно списывает резерв
15. Notification Worker отправляет уведомление
16. Cart Service очищает корзину
```

Если товара нет:

```text
inventory.reservation_failed
    |
    v
Order Service: failed
```

Если оплата не прошла:

```text
payment.failed
    |
    +--> Order Service: cancelled
    |
    +--> Inventory Service: освободить резерв
```

Это упрощённая Saga.

# Как организовать Polyrepo

Создай отдельные репозитории:

```text
storeforge-auth-service
storeforge-catalog-service
storeforge-cart-service
storeforge-order-service
storeforge-inventory-service
storeforge-payment-service
storeforge-notification-service
storeforge-infrastructure
```

Позже:

```text
storeforge-review-service
storeforge-contracts
```

Каждый сервис должен иметь примерно одинаковую структуру:

```text
storeforge-order-service/
├── src/
│   └── order_service/
│       ├── main.py
│       ├── api/
│       ├── models/
│       ├── schemas/
│       ├── repositories/
│       ├── services/
│       ├── consumers/
│       ├── messaging/
│       ├── database/
│       └── config.py
├── migrations/
├── tests/
├── Dockerfile
├── alembic.ini
├── pyproject.toml
├── .env.example
└── README.md
```

Отдельный репозиторий инфраструктуры:

```text
storeforge-infrastructure/
├── compose.yml
├── postgres/
├── rabbitmq/
├── nginx/
├── monitoring/
├── helm/
└── README.md
```

# Общий стек

Для всех FastAPI-сервисов:

```text
Python 3.13
FastAPI
Pydantic
pydantic-settings
SQLAlchemy 2.x async
asyncpg
Alembic
httpx
aio-pika
Redis
pytest
ruff
mypy
```

Инфраструктура:

```text
PostgreSQL
Redis
RabbitMQ
Docker
Docker Compose
Nginx или Traefik
```

Наблюдаемость позднее:

```text
OpenTelemetry
OpenTelemetry Collector
Prometheus
Grafana
Tempo или Jaeger
Loki или OpenSearch
```

# План разработки

## Этап 1. Спроектировать систему

До написания кода зафиксируй:

* список сервисов;
* ответственность каждого сервиса;
* какие данные кому принадлежат;
* HTTP API;
* события RabbitMQ;
* процесс оформления заказа;
* статусы заказа;
* структуру репозиториев.

Создай документ:

```text
storeforge-architecture.md
```

В нём опиши:

```text
Сервис
Назначение
База данных
HTTP endpoints
Публикуемые события
Получаемые события
```

Не начинай сразу со всех сервисов.

---

## Этап 2. Создать Infrastructure Repository

Сначала подними локальную инфраструктуру:

```text
PostgreSQL
Redis
RabbitMQ
RabbitMQ Management UI
API Gateway
Mailpit
```

Первый `compose.yml` может пока не запускать сами приложения.

Проверь:

```text
PostgreSQL доступен
Redis принимает команды
RabbitMQ UI открывается
```

---

## Этап 3. Создать шаблон микросервиса

Сделай один эталонный FastAPI-сервис:

```text
storeforge-service-template
```

В нём должны быть:

```http
GET /health/live
GET /health/ready
```

Также:

* конфигурация через environment;
* подключение к PostgreSQL;
* Alembic;
* JSON-логи;
* обработка ошибок;
* Dockerfile;
* базовые тесты.

Затем используй его как основу для остальных сервисов.

Не обязательно превращать его в сложный генератор. Достаточно Git template repository.

---

## Этап 4. Auth Service

Сделай:

* модель пользователя;
* регистрацию;
* логин;
* JWT;
* refresh-токены;
* роли;
* endpoint текущего пользователя;
* миграции;
* unit- и integration-тесты.

После этого через Gateway должно работать:

```text
POST /api/v1/auth/register
POST /api/v1/auth/login
GET  /api/v1/users/me
```

---

## Этап 5. Catalog Service

Реализуй:

* категории;
* товары;
* цены;
* CRUD для администратора;
* чтение каталога;
* фильтрацию;
* пагинацию;
* кеширование товара в Redis.

После этого пользователь должен:

```text
зарегистрироваться
авторизоваться
увидеть каталог
```

---

## Этап 6. Cart Service

Реализуй корзину в Redis:

```text
cart:{user_id}
```

Добавь:

* добавить товар;
* изменить количество;
* удалить товар;
* получить корзину;
* очистить корзину.

Cart Service вызывает Catalog Service, чтобы получить актуальные данные товара.

На этом этапе уже получится простой интернет-магазин без оформления заказа.

---

## Этап 7. Order Service

Начни с простого варианта без RabbitMQ:

```text
создать заказ
сохранить состав заказа
посчитать сумму
показать список заказов
```

Сохраняй снимок товара:

```text
product_id
product_name
unit_price
quantity
```

Не получай старую цену из Catalog Service при просмотре заказа.

Добавь защиту от двойного создания заказа через:

```http
Idempotency-Key
```

---

## Этап 8. Inventory Service

Сделай:

* остатки;
* поступление товара;
* резерв;
* освобождение;
* списание;
* защиту от отрицательного остатка.

Сначала можно проверить API напрямую.

Потом подключи RabbitMQ:

```text
inventory.reserve
inventory.reserved
inventory.reservation_failed
```

---

## Этап 9. Связать Order и Inventory

Order Service при создании заказа:

```text
1. сохраняет заказ;
2. публикует inventory.reserve;
3. ждёт ответ через событие;
4. обновляет статус.
```

На этом этапе внедри:

* обработку повторных сообщений;
* `processed_events`;
* `ack/nack`;
* retry;
* dead-letter queue.

---

## Этап 10. Payment Service

Сначала сделай mock-оплату.

Процесс:

```text
Order Service
    -> payment.create

Payment Service
    -> payment.created
    -> payment.succeeded или payment.failed
```

Свяжи результат с Order Service.

При неуспешной оплате:

```text
Order Service отменяет заказ
Inventory Service освобождает резерв
```

После этого основной бизнес-процесс StoreForge готов.

---

## Этап 11. Transactional Outbox

Когда обычная отправка сообщений заработает, добавь Outbox Pattern.

В каждой важной транзакции:

```text
изменение бизнес-данных
+
запись события в outbox_events
```

В одной SQL-транзакции.

Отдельный publisher читает outbox и отправляет сообщения в RabbitMQ.

Outbox нужен как минимум для:

```text
Order Service
Inventory Service
Payment Service
```

---

## Этап 12. Notification Service

Добавь consumer событий:

```text
user.registered
order.created
inventory.reservation_failed
payment.succeeded
payment.failed
order.cancelled
```

Сначала отправляй уведомления в Mailpit.

Потом можешь добавить Telegram.

---

## Этап 13. Тестирование

Для каждого сервиса:

```text
unit tests
integration tests
API tests
consumer tests
```

Для всей системы:

```text
end-to-end тест оформления заказа
```

Главный сценарий:

```text
создать пользователя
создать товар
установить остаток
положить товар в корзину
создать заказ
зарезервировать товар
успешно оплатить
проверить статус paid
```

Также обязательно проверь:

```text
товара недостаточно
двойное оформление заказа
повторное RabbitMQ-сообщение
ошибка оплаты
освобождение резерва
недоступность RabbitMQ
```

---

## Этап 14. CI

В каждом репозитории настрой pipeline:

```text
ruff
mypy
pytest
docker build
docker push
```

Каждый сервис собирает собственный образ:

```text
storeforge/auth-service:1.0.0
storeforge/catalog-service:1.0.0
storeforge/order-service:1.0.0
```

Для начала deploy можно делать вручную.

---

## Этап 15. Observability

Когда бизнес-логика работает, добавь:

* JSON-логи;
* `request_id`;
* `correlation_id`;
* трассировку HTTP;
* трассировку RabbitMQ;
* метрики;
* дашборды.

Главный результат — возможность увидеть весь путь заказа:

```text
POST /orders
-> Order Service
-> RabbitMQ
-> Inventory Service
-> RabbitMQ
-> Payment Service
-> Notification Service
```

в одном trace.

---

## Этап 16. Kubernetes и CD

Не переходи к Kubernetes, пока система стабильно не работает в Docker Compose.

После этого:

* Deployment для каждого сервиса;
* Service;
* Ingress;
* ConfigMap;
* Secret или ExternalSecret;
* PostgreSQL лучше оставить внешним либо поднять оператором;
* RabbitMQ Cluster Operator;
* Redis;
* readiness/liveness probes;
* resource requests/limits;
* Helm charts;
* Argo CD.

# Оптимальная последовательность репозиториев

Создавай их в таком порядке:

```text
1. storeforge-infrastructure
2. storeforge-auth-service
3. storeforge-catalog-service
4. storeforge-cart-service
5. storeforge-order-service
6. storeforge-inventory-service
7. storeforge-payment-service
8. storeforge-notification-service
9. storeforge-review-service
```

# Что не надо делать в начале

Не начинай сразу с:

* Kubernetes;
* service mesh;
* Kafka;
* Elasticsearch;
* GraphQL;
* сложной Saga framework;
* отдельной базы PostgreSQL в отдельном контейнере для каждого сервиса;
* 15 микросервисов;
* автоматического production-deploy;
* слишком сложной Clean Architecture.

Сначала должен заработать основной поток:

```text
Пользователь
→ Каталог
→ Корзина
→ Заказ
→ Резерв
→ Оплата
→ Уведомление
```

# Что сделать прямо сейчас

Начальный порядок работы:

```text
1. Создать Git-организацию StoreForge.
2. Создать storeforge-infrastructure.
3. Поднять PostgreSQL, Redis и RabbitMQ через Docker Compose.
4. Создать шаблон FastAPI-сервиса.
5. Создать auth-service.
6. Реализовать регистрацию и логин.
7. Создать catalog-service.
8. Реализовать товары и категории.
9. Подключить API Gateway.
10. Только после этого переходить к корзине и заказам.
```

Главная архитектурная идея: **Order Service управляет оформлением заказа, Inventory Service владеет остатками, Payment Service владеет платежами, Catalog Service владеет товарами. Ни один сервис не читает напрямую чужую базу данных.**

# API Gateway

Go-сервис между Quote Service и Mobile App. Единственная точка входа для мобильного приложения.

## Задачи

- Превращать gRPC-вызовы Quote Service в REST API
- Кэшировать ответы для выдерживания polling от мобилки
- Проверять JWT-токены, ограничивать частоту запросов

## Точка входа и конфиг

| Путь | Что внутри |
|------|-----------|
| `cmd/main.go` | Инициализация, запуск HTTP-сервера |
| `configs/config.yaml` | Порт, адрес Quote Service, JWT secret |
| `go.mod` | Зависимости |
| `Makefile` | Запуск, линт, тесты |

## Handlers

| Путь | Что внутри |
|------|-----------|
| `internal/handler/quote.go` | Котировки: getQuote, batchQuotes, getHistory |
| `internal/handler/portfolio.go` | Портфель: getPortfolio, getPositions |
| `internal/handler/order.go` | Заявки: placeOrder, cancelOrder, getOrders |

## Middleware и Cache

| Путь | Что внутри |
|------|-----------|
| `internal/cache/local.go` | In-memory кэш с TTL |
| `internal/middleware/auth.go` | JWT валидация |
| `internal/middleware/ratelimit.go` | Rate limiting |
| `internal/middleware/logging.go` | Логирование запросов |

## gRPC Client

| Путь | Что внутри |
|------|-----------|
| `internal/grpc/client.go` | Подключение к Quote Service, пул соединений |

## Слои

| Слой | Пакет | Что внутри |
|------|-------|-----------|
| Handler | `internal/handler` | HTTP handlers, маппинг в gRPC |
| Cache | `internal/cache` | In-memory кэш с TTL |
| Middleware | `internal/middleware` | Auth, rate limit, logging |
| gRPC Client | `internal/grpc` | Подключение к Quote Service |

## Кэширование

| Данные | TTL | Почему |
|--------|-----|--------|
| Котировка | 1 секунда | Меняется часто |
| История свечей | 30 секунд | История стабильна |
| Портфель | 5 секунд | Меняется только после сделок |

Кэш in-memory. При масштабировании можно вынести в общий Redis.

## REST API — Quotes

| Метод | Путь | Описание | Ответ |
|-------|------|----------|-------|
| GET | `/api/v1/quotes/{ticker}` | Текущая котировка | Quote |
| POST | `/api/v1/quotes/batch` | Пакет котировок | Quote[] |
| GET | `/api/v1/quotes/{ticker}/history` | История свечей | Candle[] |

Параметры history:

| Параметр | Тип | Пример |
|----------|-----|--------|
| interval | string | `1M`, `5M`, `1H`, `1D` |
| from | ISO 8601 | `2024-01-01T00:00:00Z` |
| to | ISO 8601 | `2024-01-02T00:00:00Z` |

## REST API — Portfolio

| Метод | Путь | Описание | Ответ |
|-------|------|----------|-------|
| GET | `/api/v1/portfolio` | Общая стоимость, P&L | Portfolio |
| GET | `/api/v1/portfolio/positions` | Список позиций | Position[] |

## REST API — Orders

| Метод | Путь | Описание | Тело запроса | Ответ |
|-------|------|----------|-------------|-------|
| POST | `/api/v1/orders` | Разместить заявку | OrderRequest | OrderResponse |
| DELETE | `/api/v1/orders/{id}` | Отменить заявку | — | 204 |
| GET | `/api/v1/orders` | Список заявок | status (опционально) | Order[] |

Тело POST /orders:

| Поле | Тип | Пример |
|------|-----|--------|
| ticker | string | `AAPL` |
| side | string | `buy`, `sell` |
| quantity | int | `10` |
| price | float | `150.00` |
| order_type | string | `market`, `limit` |

## REST API — User

| Метод | Путь | Описание | Ответ |
|-------|------|----------|-------|
| GET | `/api/v1/user` | Профиль | User |

## Формат ответов

### Quote

| Поле | Тип |
|------|-----|
| ticker | string |
| price | float |
| volume | int |
| timestamp | int64 |
| bid | float |
| ask | float |

### Candle

| Поле | Тип |
|------|-----|
| ticker | string |
| open | float |
| high | float |
| low | float |
| close | float |
| volume | int |
| timestamp | int64 |

### Position

| Поле | Тип |
|------|-----|
| ticker | string |
| quantity | int |
| avg_price | float |
| current_price | float |
| pnl | float |
| pnl_percent | float |

### Order

| Поле | Тип |
|------|-----|
| id | string |
| ticker | string |
| side | string |
| quantity | int |
| price | float |
| order_type | string |
| status | string |
| created_at | string |

## Middleware

| Middleware | Что делает |
|------------|-----------|
| Auth | Проверяет JWT из Authorization. Без токена — 401 |
| Rate Limit | 60/мин котировки, 10/мин ордера. Превышение — 429 |
| Logging | Метод, статус, время выполнения |

## Ошибки

| HTTP код | Когда |
|----------|-------|
| 400 | Неверный запрос |
| 401 | Нет или просрочен JWT |
| 404 | Тикер не найден |
| 429 | Rate limit превышен |
| 502 | Quote Service недоступен |

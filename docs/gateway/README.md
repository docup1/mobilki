# API Gateway

Go-сервис между Quote Service и Mobile App. Единственная точка входа для мобильного приложения.

## Задачи

- Превращать gRPC-вызовы Quote Service в REST API
- Кэшировать ответы, чтобы выдерживать частый polling от мобилки
- Проверять JWT-токены
- Ограничивать частоту запросов

## Структура проекта

| Путь | Что внутри |
| --- | --- |
| `cmd/main.go` | Точка входа, инициализация зависимостей, запуск HTTP-сервера |
| `internal/handler/quote.go` | HTTP handlers для котировок: getQuote, batchQuotes, getHistory |
| `internal/handler/portfolio.go` | HTTP handlers для портфеля: getPortfolio, getPositions |
| `internal/handler/order.go` | HTTP handlers для заявок: placeOrder, cancelOrder, getOrders |
| `internal/cache/local.go` | In-memory кэш с TTL |
| `internal/middleware/auth.go` | JWT валидация из заголовка Authorization |
| `internal/middleware/ratelimit.go` | Rate limiting: 60/мин котировки, 10/мин ордера |
| `internal/middleware/logging.go` | Логирование метода, статуса, времени выполнения |
| `internal/grpc/client.go` | gRPC клиент к Quote Service, пул соединений |
| `configs/config.yaml` | Конфигурация: порт, адрес Quote Service, JWT secret |
| `go.mod` | Зависимости |
| `Makefile` | Команды: запуск, линт, тесты |

## Слои

| Слой | Пакет | Что внутри |
| --- | --- | --- |
| Handler | `internal/handler` | HTTP handlers, маппинг запросов в gRPC |
| Cache | `internal/cache` | In-memory кэш с TTL |
| Middleware | `internal/middleware` | Auth, rate limit, logging |
| gRPC Client | `internal/grpc` | Подключение к Quote Service |

## Кэширование

Gateway хранит в памяти последние котировки с TTL. Когда приходит запрос — сначала проверяет кэш. Только при промахе идёт в Quote Service по gRPC.

| Данные | TTL | Почему |
| --- | --- | --- |
| Котировка | 1 секунда | Котировки меняются часто, но не чаще раза в секунду |
| История свечей | 30 секунд | История не меняется так быстро |
| Портфель | 5 секунд | Портфель меняется только после сделок |

Кэш in-memory, без внешней Redis. Это проще и быстрее для одного инстанса Gateway. Если нужно масштабировать — можно добавить общий Redis.

## REST API

### Quotes

| Метод | Путь | Описание | Параметры | Ответ |
| --- | --- | --- | --- | --- |
| GET | `/api/v1/quotes/{ticker}` | Текущая котировка по тикеру | ticker в пути | Quote |
| POST | `/api/v1/quotes/batch` | Пакет котировок для списка | tickers: string\[\] | Quote\[\] |
| GET | `/api/v1/quotes/{ticker}/history` | История свечей | interval, from, to в query | Candle\[\] |

Параметры history:

| Параметр | Тип | Описание | Пример |
| --- | --- | --- | --- |
| interval | string | Интервал свечи: 1M, 5M, 15M, 1H, 1D, 1W | `1D` |
| from | string | Начало периода, ISO 8601 | `2024-01-01T00:00:00Z` |
| to | string | Конец периода, ISO 8601 | `2024-01-02T00:00:00Z` |

### Portfolio

| Метод | Путь | Описание | Ответ |
| --- | --- | --- | --- |
| GET | `/api/v1/portfolio` | Общая стоимость портфеля, P&L за день | Portfolio |
| GET | `/api/v1/portfolio/positions` | Список открытых позиций | Position\[\] |

### Orders

| Метод | Путь | Описание | Тело запроса | Ответ |
| --- | --- | --- | --- | --- |
| POST | `/api/v1/orders` | Разместить заявку | OrderRequest | OrderResponse |
| DELETE | `/api/v1/orders/{id}` | Отменить заявку | — | 204 |
| GET | `/api/v1/orders` | Список заявок | status в query (опционально) | Order\[\] |

Тело POST /orders:

| Поле | Тип | Описание | Пример |
| --- | --- | --- | --- |
| ticker | string | Тикер инструмента | `AAPL` |
| side | string | Направление: buy или sell | `buy` |
| quantity | int | Количество лотов | `10` |
| price | float | Цена (для limit) | `150.00` |
| order_type | string | Тип: market или limit | `limit` |

### User

| Метод | Путь | Описание | Ответ |
| --- | --- | --- | --- |
| GET | `/api/v1/user` | Профиль пользователя | User |

## Формат ответов

### Quote

| Поле | Тип | Описание |
| --- | --- | --- |
| ticker | string | Тикер |
| price | float | Последняя цена |
| volume | int | Объём |
| timestamp | int64 | Unix timestamp в миллисекундах |
| bid | float | Цена покупки |
| ask | float | Цена продажи |

### Candle

| Поле | Тип | Описание |
| --- | --- | --- |
| ticker | string | Тикер |
| open | float | Цена открытия |
| high | float | Максимум |
| low | float | Минимум |
| close | float | Цена закрытия |
| volume | int | Объём |
| timestamp | int64 | Unix timestamp |

### Position

| Поле | Тип | Описание |
| --- | --- | --- |
| ticker | string | Тикер |
| quantity | int | Количество |
| avg_price | float | Средняя цена покупки |
| current_price | float | Текущая цена |
| pnl | float | Прибыль/убыток |
| pnl_percent | float | Прибыль/убыток в процентах |

### Order

| Поле | Тип | Описание |
| --- | --- | --- |
| id | string | ID заявки |
| ticker | string | Тикер |
| side | string | buy / sell |
| quantity | int | Количество |
| price | float | Цена |
| order_type | string | market / limit |
| status | string | new, filled, cancelled, rejected |
| created_at | string | Время создания, ISO 8601 |

## Middleware

| Middleware | Файл | Что делает |
| --- | --- | --- |
| Auth | `middleware/auth.go` | Проверяет JWT из заголовка Authorization. Без токена — 401 |
| Rate Limit | `middleware/ratelimit.go` | 60 запросов/мин для котировок, 10/мин для ордеров. Превышение — 429 |
| Logging | `middleware/logging.go` | Логирует метод, путь, статус, время выполнения |

## Обработка ошибок

| HTTP код | Когда |
| --- | --- |
| 400 | Неверный запрос — нет обязательных полей |
| 401 | Нет или просрочен JWT-токен |
| 404 | Тикер не найден |
| 429 | Rate limit превышен |
| 502 | Quote Service недоступен |

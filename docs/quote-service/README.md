# Quote Service

Go-микросервис между kernel driver и Gateway.

## Задачи

- Читать котировки из kernel driver через character device `/dev/quotes`
- Отдавать котировки по gRPC — одиночные запросы и потоковые подписки
- Публиковать обновления в Redis Streams для асинхронных потребителей

## Архитектура

Сервис построен на DDD и Clean Architecture. Зависимости направлены внутрь — к домену. Доменный слой не знает ни про kernel driver, ни про gRPC, ни про Redis.

Kernel driver вызывается напрямую из слоя инфраструктуры — через чтение character device `/dev/quotes`.

## Точка входа и конфиг

| Путь | Что внутри |
|------|-----------|
| `cmd/server/main.go` | Инициализация зависимостей, запуск сервера |
| `configs/config.yaml` | Адрес Redis, порт gRPC, путь к device |
| `go.mod` | Зависимости |
| `Makefile` | Генерация proto, запуск, тесты |

## Domain

| Путь | Что внутри |
|------|-----------|
| `internal/domain/quote.go` | Модель Quote, value-объекты |
| `internal/domain/ticker.go` | Агрегат Ticker |
| `internal/domain/errors.go` | Доменные ошибки |
| `internal/domain/events.go` | Доменные события |

## Use Cases

| Путь | Что внутри |
|------|-----------|
| `internal/usecase/get_quote.go` | Получить последнюю котировку |
| `internal/usecase/subscribe.go` | Подписка на поток котировок |
| `internal/usecase/ports.go` | Интерфейсы QuoteRepository и QuotePublisher |

## Infrastructure

| Путь | Что внутри |
|------|-----------|
| `internal/infrastructure/driver/device.go` | Открытие `/dev/quotes`, read/poll |
| `internal/infrastructure/driver/adapter.go` | Парсинг байтов в Quote |
| `internal/infrastructure/redis/publisher.go` | Публикация в Redis Streams |
| `internal/infrastructure/grpc/server.go` | gRPC сервер, вызов use cases |

## Delivery

| Путь | Что внутри |
|------|-----------|
| `internal/delivery/proto/quote.proto` | Protobuf контракт QuoteService |

## Слои

| Слой | Пакет | От чего не зависит |
|------|-------|-------------------|
| Domain | `internal/domain` | Ни от чего |
| Use Cases | `internal/usecase` | От инфраструктуры и delivery |
| Infrastructure | `internal/infrastructure` | От delivery |
| Delivery | `internal/delivery` | — |

## Доменная модель

| Тип | Файл | Что это |
|-----|------|---------|
| Quote | `domain/quote.go` | Котировка: тикер, цена, объём, время, bid, ask |
| TickerSymbol | `domain/quote.go` | Строка-тикер |
| Price | `domain/quote.go` | Типизированная цена |
| Volume | `domain/quote.go` | Типизированный объём |
| Ticker | `domain/ticker.go` | Агрегат: текущая котировка + история |
| QuoteUpdated | `domain/events.go` | Событие для публикации в Redis |

Агрегат Ticker — чистая функция. ApplyQuote возвращает новое состояние, не меняя исходное.

## Use Cases

| Use Case | Что делает |
|----------|-----------|
| GetQuote | Получает последнюю котировку по тикеру |
| SubscribeToQuotes | Подписывается на поток и публикует в Redis |

## Порты

| Порт | Методы | Кто реализует |
|------|--------|--------------|
| QuoteRepository | GetLatest, Subscribe | Device driver adapter |
| QuotePublisher | Publish | Redis publisher |

## Взаимодействие с kernel driver

| Операция | Системный вызов | Что происходит |
|----------|----------------|---------------|
| Открытие | `open("/dev/quotes", O_RDONLY)` | Получаем файловый дескриптор |
| Ожидание | `poll(fd, POLLIN)` | Блокируемся до появления данных |
| Чтение | `read(fd, buffer)` | Забираем байты из ring buffer ядра |
| Закрытие | `close(fd)` | Освобождаем дескриптор |

Kernel driver будит Quote Service через waitqueue при каждой новой котировке. Quote Service не опрашивает — получает данные сразу.

## Режимы отдачи котировок

| Режим | Для кого | Как работает |
|-------|----------|-------------|
| gRPC Unary | Gateway | Одиночный запрос — ответ |
| gRPC Streaming | Gateway | Подписка на поток обновлений |
| Redis Streams | Аналитика, архив | Асинхронная запись с replay |

## Ошибки

| Ошибка | Когда возникает |
|--------|----------------|
| QuoteNotFound | Тикер не найден в буфере |
| DriverOffline | `/dev/quotes` недоступен — модуль не загружен |
| InvalidTicker | Пустой или некорректный тикер |

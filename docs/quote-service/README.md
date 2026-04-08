# Quote Service

Go-микросервис между kernel driver и Gateway.

## Задачи

- Читать котировки из kernel driver через character device `/dev/quotes`
- Отдавать котировки по gRPC — одиночные запросы и потоковые подписки
- Публиковать обновления в Redis Streams для асинхронных потребителей

## Архитектура

Сервис построен на DDD и Clean Architecture. Зависимости направлены внутрь — к домену. Это значит, что доменный слой не знает ни про kernel driver, ни про gRPC, ни про Redis. Он знает только про котировки, тикеры и события.

Kernel driver вызывается напрямую из слоя инфраструктуры — через открытие и чтение character device `/dev/quotes`.

## Структура проекта

| Путь | Что внутри |
|------|-----------|
| `cmd/server/main.go` | Точка входа, инициализация зависимостей, запуск сервера |
| `internal/domain/quote.go` | Модель Quote, value-объекты TickerSymbol, Price, Volume |
| `internal/domain/ticker.go` | Агрегат Ticker с методом ApplyQuote |
| `internal/domain/errors.go` | Доменные ошибки: QuoteNotFound, DriverOffline, InvalidTicker |
| `internal/domain/events.go` | Доменное событие QuoteUpdated |
| `internal/usecase/get_quote.go` | UseCase: получить последнюю котировку |
| `internal/usecase/subscribe.go` | UseCase: подписка на поток котировок |
| `internal/usecase/ports.go` | Интерфейсы QuoteRepository и QuotePublisher |
| `internal/infrastructure/driver/device.go` | Открытие `/dev/quotes`, read/poll |
| `internal/infrastructure/driver/adapter.go` | Адаптер байтов из device в доменные объекты Go |
| `internal/infrastructure/redis/publisher.go` | Публикация событий в Redis Streams |
| `internal/infrastructure/grpc/server.go` | gRPC сервер, делегирует use cases |
| `internal/delivery/proto/quote.proto` | Protobuf контракт QuoteService |
| `configs/config.yaml` | Конфигурация: адрес Redis, порт gRPC, путь к device |
| `go.mod` | Зависимости |
| `Makefile` | Команды: генерация proto, запуск, тесты |

## Слои

| Слой | Пакет | Что внутри | От чего не зависит |
|------|-------|-----------|-------------------|
| Domain | `internal/domain` | Quote, Ticker, ошибки, события | Ни от чего |
| Use Cases | `internal/usecase` | GetQuote, SubscribeToQuotes, интерфейсы портов | От инфраструктуры и delivery |
| Infrastructure | `internal/infrastructure` | Device driver adapter, Redis, gRPC server | От delivery |
| Delivery | `internal/delivery` | Protobuf файлы | — |

## Доменная модель

| Тип | Файл | Что это | Зачем |
|-----|------|---------|-------|
| Quote | `domain/quote.go` | Котировка: тикер, цена, объём, время, bid, ask | Основная единица данных |
| TickerSymbol | `domain/quote.go` | Строка-тикер | Чтобы не перепутать с ценой |
| Price | `domain/quote.go` | Цена | Типизированная цена |
| Volume | `domain/quote.go` | Объём | Типизированный объём |
| Ticker | `domain/ticker.go` | Агрегат: текущая котировка + история | Собирает котировки по тикеру |
| QuoteUpdated | `domain/events.go` | Событие обновления котировки | Для публикации в Redis |

Агрегат Ticker — чистая функция. Метод ApplyQuote принимает новую котировку и возвращает обновлённое состояние, не меняя исходное. Это делает домен детерминированным и легко тестируемым.

## Use Cases

| Use Case | Файл | Что делает |
|----------|------|-----------|
| GetQuote | `usecase/get_quote.go` | Получает последнюю котировку по тикеру |
| SubscribeToQuotes | `usecase/subscribe.go` | Подписывается на поток котировок и публикует в Redis |

## Порты (интерфейсы)

| Порт | Файл | Методы | Кто реализует |
|------|------|--------|--------------|
| QuoteRepository | `usecase/ports.go` | GetLatest, Subscribe | Device driver adapter |
| QuotePublisher | `usecase/ports.go` | Publish | Redis publisher |

## Взаимодействие с kernel driver

Quote Service работает с kernel driver как с обычным файлом устройства.

| Операция | Системный вызов | Что происходит |
|----------|----------------|---------------|
| Открытие | `open("/dev/quotes", O_RDONLY)` | Получаем файловый дескриптор |
| Ожидание | `poll(fd, POLLIN)` | Блокируемся до появления данных |
| Чтение | `read(fd, buffer)` | Забираем байты котировки из ring buffer ядра |
| Закрытие | `close(fd)` | Освобождаем дескриптор |

Kernel driver будит Quote Service через waitqueue каждый раз, когда в ring buffer появляется новая котировка. Это значит, что Quote Service не опрашивает драйвер — он получает данные сразу, без задержек.

Формат данных в device — бинарный: заголовок с длиной и типом, далее сериализованная котировка. Adapter парсит байты в доменный объект Quote.

## Режимы отдачи котировок

| Режим | Для кого | Как работает |
|-------|----------|-------------|
| gRPC Unary | Gateway | Одиночный запрос — ответ. Gateway вызывает при cache miss |
| gRPC Streaming | Gateway | Подписка на поток. Gateway подписывается при старте и получает все обновления |
| Redis Streams | Аналитика, архив | Асинхронная запись. Можно перечитать историю, подключить consumer groups |

## Ошибки

| Ошибка | Файл | Когда возникает |
|--------|------|----------------|
| QuoteNotFound | `domain/errors.go` | Тикер не найден в буфере |
| DriverOffline | `domain/errors.go` | `/dev/quotes` недоступен — модуль не загружен |
| InvalidTicker | `domain/errors.go` | Пустой или некорректный тикер |

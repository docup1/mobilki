# Quote Service

Go-микросервис между C-драйвером и Gateway.

## Задачи

- Читать котировки из C-драйвера через CGO
- Отдавать котировки по gRPC — одиночные запросы и потоковые подписки
- Публиковать обновления в Redis Streams для асинхронных потребителей

## Архитектура

Сервис построен на DDD и Clean Architecture. Зависимости направлены внутрь — к домену. Это значит, что доменный слой не знает ни про C-драйвер, ни про gRPC, ни про Redis. Он знает только про котировки, тикеры и события.

C-драйвер вызывается напрямую из слоя инфраструктуры — без лишних прослоек. Адаптер преобразует C-структуры в доменные объекты Go.

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
| `internal/infrastructure/driver/c_driver.go` | CGO обёртка над C-библиотекой |
| `internal/infrastructure/driver/adapter.go` | Адаптер C-структур в доменные объекты Go |
| `internal/infrastructure/redis/publisher.go` | Публикация событий в Redis Streams |
| `internal/infrastructure/grpc/server.go` | gRPC сервер, делегирует use cases |
| `internal/delivery/proto/quote.proto` | Protobuf контракт QuoteService |
| `configs/config.yaml` | Конфигурация: адрес Redis, порт gRPC |
| `go.mod` | Зависимости |
| `Makefile` | Команды: генерация proto, запуск, тесты |

## Слои

| Слой | Пакет | Что внутри | От чего не зависит |
|------|-------|-----------|-------------------|
| Domain | `internal/domain` | Quote, Ticker, ошибки, события | Ни от чего |
| Use Cases | `internal/usecase` | GetQuote, SubscribeToQuotes, интерфейсы портов | От инфраструктуры и delivery |
| Infrastructure | `internal/infrastructure` | CGO-адаптер, Redis, gRPC server | От delivery |
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
| QuoteRepository | `usecase/ports.go` | GetLatest, Subscribe | CGO-адаптер |
| QuotePublisher | `usecase/ports.go` | Publish | Redis publisher |

## Режимы отдачи котировок

| Режим | Для кого | Как работает |
|-------|----------|-------------|
| gRPC Unary | Gateway | Одиночный запрос — ответ. Gateway вызывает при cache miss |
| gRPC Streaming | Gateway | Подписка на поток. Gateway подписывается при старте и получает все обновления |
| Redis Streams | Аналитика, архив | Асинхронная запись. Можно перечитать историю, подключить consumer groups |

## Ошибки

| Ошибка | Файл | Когда возникает |
|--------|------|----------------|
| QuoteNotFound | `domain/errors.go` | Тикер не найден в драйвере |
| DriverOffline | `domain/errors.go` | C-драйвер не инициализировался |
| InvalidTicker | `domain/errors.go` | Пустой или некорректный тикер |

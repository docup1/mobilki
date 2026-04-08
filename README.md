# Документация

Архитектура системы котировок акций. Два микросервиса и мобильное приложение.

## Структура

- [Обзор](docs/overview.md) — из чего состоит система и как компоненты общаются
- [Kernel Driver](docs/kernel-driver/README.md) — модуль ядра Linux, системный поток
- [Quote Service](docs/quote-service/README.md) — Go-сервис, читает котировки из driver
- [Gateway](docs/gateway/README.md) — REST API между Quote Service и мобилкой
- [Mobile App](docs/mobile-app/README.md) — React Native, аналог Т-Инвестиций

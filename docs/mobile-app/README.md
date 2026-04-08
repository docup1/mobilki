# Mobile App

React Native приложение, аналог Т-Инвестиций.

## Архитектура

Feature-Sliced Design (FSD). Импорт только снизу вверх.

| Слой | Пакет | Может импортировать |
|------|-------|-------------------|
| shared | `src/shared` | Ничего из бизнес-логики |
| entities | `src/entities` | shared |
| features | `src/features` | entities, shared |
| widgets | `src/widgets` | features, entities, shared |
| pages | `src/pages` | widgets, features, entities, shared |

## Точка входа

| Путь | Что внутри |
|------|-----------|
| `app/providers/QueryClientProvider.tsx` | Провайдер React Query |
| `app/providers/NavigationProvider.tsx` | Провайдер навигации |
| `app/providers/ThemeProvider.tsx` | Провайдер темы |
| `app/App.tsx` | Корневой компонент |
| `app/index.tsx` | Точка входа |

## Shared — API

| Путь | Что внутри |
|------|-----------|
| `src/shared/api/gateway.ts` | REST клиент на axios |
| `src/shared/api/types.ts` | DTO: Quote, Candle, Order... |
| `src/shared/api/interceptors.ts` | Auth и retry |
| `src/shared/config/endpoints.ts` | URL Gateway |

## Shared — UI и утилиты

| Путь | Что внутри |
|------|-----------|
| `src/shared/ui/Button/` | Кнопка |
| `src/shared/ui/Chart/` | График цен |
| `src/shared/ui/QuoteCard/` | Карточка котировки |
| `src/shared/ui/OrderBook/` | Стакан |
| `src/shared/lib/format.ts` | Форматирование цен, дат |
| `src/shared/lib/constants.ts` | Константы |

## Entities — Quote

| Путь | Что внутри |
|------|-----------|
| `src/entities/quote/model/types.ts` | Типы Quote, Ticker |
| `src/entities/quote/model/hooks.ts` | useQuote, useQuotes, useQuoteHistory |
| `src/entities/quote/ui/QuoteDisplay.tsx` | Отображение котировки |

## Entities — Portfolio, Order, User

| Путь | Что внутри |
|------|-----------|
| `src/entities/portfolio/model/types.ts` | Типы Portfolio, Position |
| `src/entities/portfolio/model/hooks.ts` | usePortfolio, usePositions |
| `src/entities/portfolio/api/portfolioApi.ts` | Маппинг DTO → domain |
| `src/entities/order/model/types.ts` | Типы Order |
| `src/entities/order/api/orderApi.ts` | Маппинг DTO → domain |
| `src/entities/user/model/types.ts` | Типы User |

## Features

| Путь | Что внутри |
|------|-----------|
| `src/features/buy-sell/ui/BuySellSheet.tsx` | Модалка покупки/продажи |
| `src/features/buy-sell/ui/OrderForm.tsx` | Форма ордера |
| `src/features/buy-sell/model/usePlaceOrder.ts` | Хук размещения ордера |
| `src/features/quote-search/ui/QuoteSearch.tsx` | Поиск с автодополнением |
| `src/features/quote-search/model/useQuoteSearch.ts` | Хук поиска |
| `src/features/chart-timeframe/ui/TimeframeSelector.tsx` | Переключатель таймфрейма |
| `src/features/chart-timeframe/model/useTimeframe.ts` | Хук выбора периода |

## Widgets

| Путь | Что внутри |
|------|-----------|
| `src/widgets/quote-detail/ui/QuoteDetailWidget.tsx` | Полный экран котировки |
| `src/widgets/portfolio-summary/ui/PortfolioWidget.tsx` | Виджет портфеля |
| `src/widgets/market-overview/ui/MarketWidget.tsx` | Сводка рынка |
| `src/widgets/order-book/ui/OrderBookWidget.tsx` | Стакан |

## Pages

| Путь | Что внутри |
|------|-----------|
| `src/pages/main/MainScreen.tsx` | Главная |
| `src/pages/quote-detail/QuoteDetailScreen.tsx` | Детали котировки |
| `src/pages/portfolio/PortfolioScreen.tsx` | Портфель |
| `src/pages/search/SearchScreen.tsx` | Поиск |
| `src/pages/profile/ProfileScreen.tsx` | Профиль |

## Навигация

| Путь | Что внутри |
|------|-----------|
| `navigation/RootNavigator.tsx` | Stack + Tab навигация |
| `navigation/types.ts` | Типы экранов |

## Табы

| Таб | Файл |
|-----|------|
| Главная | `pages/main/MainScreen.tsx` |
| Поиск | `pages/search/SearchScreen.tsx` |
| Портфель | `pages/portfolio/PortfolioScreen.tsx` |
| Профиль | `pages/profile/ProfileScreen.tsx` |

## Stack экраны

| Экран | Файл | Как открывается |
|-------|------|----------------|
| QuoteDetail | `pages/quote-detail/QuoteDetailScreen.tsx` | По тапу на котировку |
| OrderSheet | `features/buy-sell/ui/BuySellSheet.tsx` | Модалка по кнопке покупки |

## Polling

| Данные | Интервал | Хук |
|--------|----------|-----|
| Одна котировка | 1 секунда | useQuote |
| Пакет котировок | 2 секунды | useQuotes |
| История свечей | 30 секунд | useQuoteHistory |

Хук useVisibleQuote отключает polling, когда экран не в фокусе.

## State management

| Тип | Инструмент | Где | Что хранит |
|-----|-----------|-----|-----------|
| Server state | React Query | `entities/*/model/hooks.ts` | Котировки, портфель, ордера |
| Client state | Zustand | `entities/*/model/store.ts` | Выбранная позиция, фильтры |

## API клиент

| Метод | Endpoint |
|-------|----------|
| getQuote | GET /quotes/{ticker} |
| getQuotes | POST /quotes/batch |
| getQuoteHistory | GET /quotes/{ticker}/history |
| getPortfolio | GET /portfolio |
| getPositions | GET /portfolio/positions |
| placeOrder | POST /orders |
| cancelOrder | DELETE /orders/{id} |
| getOrders | GET /orders |
| getProfile | GET /user |

## Экран котировки

| Блок | Компонент | Данные |
|------|-----------|--------|
| Шапка с ценой | QuoteHeader | useVisibleQuote |
| График | PriceChart | useQuoteHistory |
| Статистика | KeyStats | useVisibleQuote |
| Стакан | OrderBookWidget | polling 2 секунды |
| Кнопка | BuySellButton | открывает модалку |

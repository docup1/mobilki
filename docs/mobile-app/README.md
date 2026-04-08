# Mobile App

React Native приложение, аналог Т-Инвестиций.

## Архитектура

Используется Feature-Sliced Design (FSD). Код разделён на слои, импорт разрешён только снизу вверх. Это значит, что фичи не зависят друг от друга, а экраны зависят от виджетов и фич.

| Слой | Пакет | Что внутри | Может импортировать |
| --- | --- | --- | --- |
| shared | `src/shared` | UI-компоненты, API-клиент, утилиты | Ничего из бизнес-логики |
| entities | `src/entities` | Бизнес-сущности: модели, хуки, API | shared |
| features | `src/features` | Действия: купить/продать, поиск | entities, shared |
| widgets | `src/widgets` | Композитные блоки страниц | features, entities, shared |
| pages | `src/pages` | Экраны приложения | widgets, features, entities, shared |

## Структура проекта

| Путь | Что внутри |
| --- | --- |
| `app/providers/QueryClientProvider.tsx` | Провайдер React Query с настройками кэша |
| `app/providers/NavigationProvider.tsx` | Провайдер навигации |
| `app/providers/ThemeProvider.tsx` | Провайдер темы приложения |
| `app/App.tsx` | Корневой компонент, собирает провайдеров |
| `app/index.tsx` | Точка входа |
| `src/shared/api/gateway.ts` | Gateway REST клиент на axios |
| `src/shared/api/types.ts` | DTO: Quote, Candle, Order, Portfolio, Position, User |
| `src/shared/api/interceptors.ts` | Auth и retry интерцепторы |
| `src/shared/ui/Button/` | Кнопка |
| `src/shared/ui/Chart/` | График цен |
| `src/shared/ui/QuoteCard/` | Карточка котировки для списков |
| `src/shared/ui/OrderBook/` | Стакан |
| `src/shared/lib/format.ts` | Форматирование цен, дат, процентов |
| `src/shared/lib/constants.ts` | Константы приложения |
| `src/shared/config/endpoints.ts` | URL Gateway |
| `src/entities/quote/model/types.ts` | Типы Quote, Ticker для domain-слоя |
| `src/entities/quote/model/hooks.ts` | useQuote, useQuotes, useQuoteHistory |
| `src/entities/quote/ui/QuoteDisplay.tsx` | Отображение котировки |
| `src/entities/portfolio/model/types.ts` | Типы Portfolio, Position |
| `src/entities/portfolio/model/hooks.ts` | usePortfolio, usePositions |
| `src/entities/portfolio/api/portfolioApi.ts` | Маппинг DTO → domain |
| `src/entities/order/model/types.ts` | Типы Order |
| `src/entities/order/api/orderApi.ts` | Маппинг DTO → domain |
| `src/entities/user/model/types.ts` | Типы User |
| `src/features/buy-sell/ui/BuySellSheet.tsx` | Модалка покупки/продажи |
| `src/features/buy-sell/ui/OrderForm.tsx` | Форма ордера с валидацией |
| `src/features/buy-sell/model/usePlaceOrder.ts` | Хук размещения ордера + инвалидация кэша |
| `src/features/quote-search/ui/QuoteSearch.tsx` | Поиск с автодополнением |
| `src/features/quote-search/model/useQuoteSearch.ts` | Хук поиска по тикерам |
| `src/features/chart-timeframe/ui/TimeframeSelector.tsx` | Переключатель таймфрейма |
| `src/features/chart-timeframe/model/useTimeframe.ts` | Хук выбора периода |
| `src/widgets/quote-detail/ui/QuoteDetailWidget.tsx` | Полный экран котировки |
| `src/widgets/portfolio-summary/ui/PortfolioWidget.tsx` | Виджет портфеля на главной |
| `src/widgets/market-overview/ui/MarketWidget.tsx` | Сводка рынка |
| `src/widgets/order-book/ui/OrderBookWidget.tsx` | Стакан |
| `src/pages/main/MainScreen.tsx` | Главная: список котировок + сводка |
| `src/pages/quote-detail/QuoteDetailScreen.tsx` | Детали котировки |
| `src/pages/portfolio/PortfolioScreen.tsx` | Портфель с позициями |
| `src/pages/search/SearchScreen.tsx` | Поиск |
| `src/pages/profile/ProfileScreen.tsx` | Профиль |
| `navigation/RootNavigator.tsx` | Stack + Tab навигация |
| `navigation/types.ts` | Типы экранов для навигации |
| `app.json` | Конфиг Expo |
| `package.json` | Зависимости |
| `tsconfig.json` | Настройки TypeScript |

## Навигация

Bottom Tabs: Главная, Поиск, Портфель, Профиль. Из табов можно провалиться в детали котировки или модалку размещения ордера.

| Таб | Компонент | Файл |
| --- | --- | --- |
| Главная | MainScreen | `pages/main/MainScreen.tsx` |
| Поиск | SearchScreen | `pages/search/SearchScreen.tsx` |
| Портфель | PortfolioScreen | `pages/portfolio/PortfolioScreen.tsx` |
| Профиль | ProfileScreen | `pages/profile/ProfileScreen.tsx` |

| Stack экран | Компонент | Файл | Как открывается |
| --- | --- | --- | --- |
| QuoteDetail | QuoteDetailScreen | `pages/quote-detail/QuoteDetailScreen.tsx` | По тапу на котировку |
| OrderSheet | BuySellSheet | `features/buy-sell/ui/BuySellSheet.tsx` | Модалка по кнопке покупки |

## Real-time через polling

Котировки обновляются через REST polling. React Query опрашивает Gateway с заданным интервалом. Когда экран не в фокусе — polling останавливается.

| Данные | Интервал | Хук | Файл |
| --- | --- | --- | --- |
| Одна котировка | 1 секунда | useQuote | `entities/quote/model/hooks.ts` |
| Пакет котировок | 2 секунды | useQuotes | `entities/quote/model/hooks.ts` |
| История свечей | 30 секунд | useQuoteHistory | `entities/quote/model/hooks.ts` |

Хук useVisibleQuote оборачивает useQuote и отключает polling, когда экран не в фокусе.

## State management

| Тип данных | Инструмент | Где лежит | Что хранит |
| --- | --- | --- | --- |
| Server state | React Query | `entities/*/model/hooks.ts` | Котировки, портфель, ордера |
| Client state | Zustand | `entities/*/model/store.ts` | Выбранная позиция, фильтры UI |

После размещения ордера React Query инвалидирует кэш портфеля и позиций — данные обновляются автоматически.

## API клиент

Единый GatewayClient на основе axios. Лежит в `shared/api/gateway.ts`.

| Метод | Что делает | Файл |
| --- | --- | --- |
| getQuote | GET /quotes/{ticker} | `shared/api/gateway.ts` |
| getQuotes | POST /quotes/batch | `shared/api/gateway.ts` |
| getQuoteHistory | GET /quotes/{ticker}/history | `shared/api/gateway.ts` |
| getPortfolio | GET /portfolio | `shared/api/gateway.ts` |
| getPositions | GET /portfolio/positions | `shared/api/gateway.ts` |
| placeOrder | POST /orders | `shared/api/gateway.ts` |
| cancelOrder | DELETE /orders/{id} | `shared/api/gateway.ts` |
| getOrders | GET /orders | `shared/api/gateway.ts` |
| getProfile | GET /user | `shared/api/gateway.ts` |

Auth-интерцептор подставляет JWT-токен из secure storage. Retry-интерцептор повторяет запрос до 3 раз при 502 от Gateway.

## Экраны

| Экран | Файл | Что показывает |
| --- | --- | --- |
| Главная | `pages/main/MainScreen.tsx` | Список популярных котировок с изменением за день, сводка рынка |
| Детали котировки | `pages/quote-detail/QuoteDetailScreen.tsx` | График цены, ключевая статистика, стакан, кнопка покупки |
| Портфель | `pages/portfolio/PortfolioScreen.tsx` | Позиции с текущей ценой и P&L, общая стоимость |
| Поиск | `pages/search/SearchScreen.tsx` | Поиск по тикерам и названиям компаний |
| Профиль | `pages/profile/ProfileScreen.tsx` | Информация о пользователе, настройки |

## Экран котировки

Экран деталей котировки — основной экран приложения. Сверху — текущая цена и изменение за день. Ниже — график цены с выбором таймфрейма. Под графиком — ключевая статистика (день, объём, bid/ask). Ниже — стакан. Внизу — кнопка покупки/продажи.

Цена обновляется каждую секунду через polling. График подгружается отдельно и обновляется реже.

| Блок | Компонент | Файл | Откуда данные |
| --- | --- | --- | --- |
| Шапка с ценой | QuoteHeader | `widgets/quote-detail/ui/QuoteDetailWidget.tsx` | useVisibleQuote |
| График | PriceChart | `shared/ui/Chart/` | useQuoteHistory |
| Статистика | KeyStats | `widgets/quote-detail/ui/QuoteDetailWidget.tsx` | useVisibleQuote |
| Стакан | OrderBookWidget | `widgets/order-book/ui/OrderBookWidget.tsx` | polling каждые 2 секунды |
| Кнопка | BuySellButton | `features/buy-sell/ui/BuySellSheet.tsx` | открывает модалку |

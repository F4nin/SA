# Задача 1. Сервис финансовых графиков

## Контекст

Банк отказывается от вендорской системы из-за частых перебоев доступности и создаёт собственное приложение для получения цен, данных графиков и индикаторов по финансовым инструментам: акциям, фьючерсам, облигациям. [source:hbpavphv13ze1nuydwh3]

## Цель API

Предоставить клиентам доступ к ценовым данным, данным графиков и техническим индикаторам через GraphQL.

## Сущности

- **`ChartData`** — точка данных графика.
- **`Indicator`** — технический индикатор (базовый или кастомный).
- **`Value`** — значение индикатора на момент времени.

## Функциональные требования (операции)

### Query

1. **`getPriceByDate(ticker: String!, date: ISO8601DateTime): ChartData!`**  
   Получить цену по тикеру на определённый момент времени. Если дата не указана — вернуть **самую последнюю** цену. [source:o1idiwby0nkoswhuklgn]

2. **`getChartByPeriod(ticker: String!, timeFrame: String!, startDate: ISO8601DateTime, endDate: ISO8601DateTime, indicator: [String]): [ChartData!]!`**  
   Получить данные для графика по тикеру и таймфрейму за период. Если период не указан — вернуть данные за **последний месяц**. [source:o1idiwby0nkoswhuklgn][source:hbpavphv13ze1nuydwh3]

### Subscription

3. **`subscribeChart: ChartData!`**  
   Подписка на обновление графика в реальном времени. [source:o1idiwby0nkoswhuklgn]

## Бизнес-требования и nullability

- Временные метки передаются в формате **ISO 8601** → кастомный скаляр `ISO8601DateTime`. [source:hbpavphv13ze1nuydwh3]
- **`closePrice` опциональна**, потому что в момент просмотра свеча за выбранный таймфрейм может быть ещё не закрыта. [source:o1idiwby0nkoswhuklgn]
- **`indicators` опциональны**, потому что запросы графиков могут не включать данные индикаторов. [source:o1idiwby0nkoswhuklgn]
- Индикаторы **неоднородны**: по одним приходит одно значение, по другим — массив значений на один момент времени. Эта неоднородность покрывается списком `values` в `Indicator`. [source:hbpavphv13ze1nuydwh3]
- `Indicator` всегда содержит `id`, `title` и массив значений → `values: [Value!]!`. [source:hbpavphv13ze1nuydwh3]

## SDL-контракт

```graphql
scalar ISO8601DateTime

type ChartData {
  ticker: String!
  timestamp: ISO8601DateTime!
  lastPrice: Float!
  openPrice: Float!
  closePrice: Float
  maxPrice: Float!
  minPrice: Float!
  indicators: [Indicator]
}

type Indicator {
  id: ID!
  title: String!
  values: [Value!]!
}

type Value {
  timestamp: ISO8601DateTime!
  valueTitle: String!
  value: String!
}

type Query {
  getPriceByDate(ticker: String!, date: ISO8601DateTime): ChartData!
  getChartByPeriod(
    ticker: String!
    timeFrame: String!
    startDate: ISO8601DateTime
    endDate: ISO8601DateTime
    indicator: [String]
  ): [ChartData!]!
}

type Subscription {
  subscribeChart: ChartData!
}

schema {
  query: Query
  subscription: Subscription
}
```

---
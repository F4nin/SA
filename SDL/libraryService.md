# Задача 2. Сервис «Библиотека отзывов»

## Контекст

Компания запускает сервис, где пользователи ведут список прочитанных книг, оставляют отзывы, оценивают книги по 5-балльной шкале и подписываются на новые отзывы других читателей.

## Цель API

Реализовать CRUD-подобный API для книг и отзывов с пагинацией и real-time-подписками.

## Сущности

- **`Book`** — книга.
- **`Review`** — отзыв на книгу.
- **`User`** — пользователь-читатель.
- **`ReviewConnection` / `ReviewEdge` / `PageInfo`** — пагинация отзывов (паттерн Relay). [source:rq0d3blsdg6wbljnwk56]

## Функциональные требования (операции)

### Query

1. **`getBooksByTag(tag: String!): [Book!]!`**  
   Получить список книг по тегу (жанру). [source:rq0d3blsdg6wbljnwk56]

2. **`getBookById(id: ID!): Book`**  
   Получить книгу по идентификатору. Клиент может опционально запросить связанные отзывы (`reviews`). Возвращает `null`, если книга не найдена.

3. **`getBookReviews(bookId: ID!, first: Int, after: String): ReviewConnection!`**  
   Получить страницу отзывов по книге с пагинацией. [source:rq0d3blsdg6wbljnwk56]

### Mutation

4. **`createBook(input: CreateBookInput!): Book!`**  
   Создать книгу с защитой от дубликатов через идемпотентность. [source:v2x9z4rwb830hxam03jj]

5. **`createReview(input: CreateReviewForBookInput!): Review!`**  
   Создать отзыв на книгу. По решению аналитика также добавлен `idempotenceKey` для защиты от дублей при повторных запросах (аналогично созданию книги). [source:cbhqwicg4sd94sqtf1de]

### Subscription

6. **`reviewAdded(bookId: ID!): Review!`**  
   Подписка на новые отзывы по книге. [source:t8w0ssdgq55z5fn4nh3e]

## Бизнес-требования и nullability

- Даты создания — ISO 8601 → скаляр `ISO8601DateTime`.
- **`publishedYear` опционален**: год издания может быть неизвестен.
- **`text` опционален**: можно поставить только оценку без текста.
- **`createdAt` обязателен**: ставится сервером автоматически при создании. [source:v2x9z4rwb830hxam03jj]
- **`rating` — Int (1–5)**, не `Enum`: это числовая шкала, валидация диапазона выполняется на сервере. [source:rq0d3blsdg6wbljnwk56]
- Ссылки на сущности внутри `Input` передаются через **`ID`**, а не объектные типы — в `Input` запрещены выходные типы. [source:rq0d3blsdg6wbljnwk56]
- Список отзывов — большой, поэтому для него используется **Connection-пагинация**. [source:rq0d3blsdg6wbljnwk56]

## SDL-контракт

```graphql
scalar ISO8601DateTime

type User {
  id: ID!
  name: String!
}

type Book {
  id: ID!
  title: String!
  author: String!
  publishedYear: Int
  tags: [String!]!
  reviews: [Review!]!
}

type Review {
  id: ID!
  book: Book!
  author: User!
  rating: Int!
  text: String
  createdAt: ISO8601DateTime!
}

type ReviewEdge {
  node: Review!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type ReviewConnection {
  edges: [ReviewEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type Query {
  getBooksByTag(tag: String!): [Book!]!
  getBookById(id: ID!): Book
  getBookReviews(bookId: ID!, first: Int, after: String): ReviewConnection!
}

input CreateBookInput {
  idempotenceKey: ID!
  title: String!
  author: String!
  publishedYear: Int
  tags: [String!]!
}

input CreateReviewForBookInput {
  bookId: ID!
  authorId: ID!
  rating: Int!
  text: String
  idempotenceKey: ID!
}

type Mutation {
  createBook(input: CreateBookInput!): Book!
  createReview(input: CreateReviewForBookInput!): Review!
}

type Subscription {
  reviewAdded(bookId: ID!): Review!
}

schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}
```

---

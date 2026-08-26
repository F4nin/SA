# Задача 3. Сервис управления задачами (трекер)

## Контекст

Компания делает мини-трекер проектов и задач. Есть **проекты** и **задачи** внутри проектов.

## Цель API

Реализовать управление проектами и задачами с статусами, частичным обновлением и мягким удалением проектов.

## Сущности

- **`Project`** — проект.
- **`Task`** — задача внутри проекта.
- **`TaskStatus`** — `Enum`, закрытый список статусов задачи. [source:rq0d3blsdg6wbljnwk56]
- **`TaskConnection` / `TaskEdge` / `PageInfo`** — пагинация задач. [source:rq0d3blsdg6wbljnwk56]

## Функциональные требования (операции)

### Query

1. **`getProjectById(id: ID!, includeDeleted: Boolean = false): Project`**  
   Получить проект по `id`. По умолчанию скрывает мягко удалённые проекты. [source:px65m8ctgvusnzczwz7c]

2. **`getAllProjects(includeDeleted: Boolean = false): [Project!]!`**  
   Получить список всех проектов. По умолчанию скрывает мягко удалённые. [source:px65m8ctgvusnzczwz7c]

3. **`getTasksByStatus(projectId: ID!, status: TaskStatus!): [Task!]!`**  
   Получить список задач проекта по статусу (простой список, без пагинации).

4. **`getProjectTasks(projectId: ID!, first: Int, after: String): TaskConnection!`**  
   Получить страницу задач проекта с пагинацией (Connection). [source:rq0d3blsdg6wbljnwk56]

### Mutation

5. **`createProject(input: CreateProjectInput!): Project!`**  
   Создать проект с идемпотентностью.

6. **`createTask(input: CreateTaskInput!): Task!`**  
   Создать задачу с идемпотентностью.

7. **`updateTask(input: UpdateTaskInput!): Task!`**  
   Частично обновить задачу (статус, описание, дедлайн).

8. **`deleteProject(id: ID!): Project!`**  
   Мягко удалить проект (установить `deletedAt`). [source:px65m8ctgvusnzczwz7c]

9. **`restoreProject(id: ID!): Project!`**  
   Восстановить мягко удалённый проект (очистить `deletedAt`). [source:px65m8ctgvusnzczwz7c]

### Subscription

10. **`taskStatusChanged(projectId: ID!): Task!`**  
   Подписка на изменение статуса задачи в проекте. [source:t8w0ssdgq55z5fn4nh3e]

## Бизнес-требования и nullability

- **`TaskStatus` — `Enum`**: закрытый список (`BACKLOG`, `TODO`, `IN_PROGRESS`, `DONE`, `CANCELLED`), не меняется без изменения схемы. [source:rq0d3blsdg6wbljnwk56]
- **`createdAt` обязателен** (ставится сервером); **`updatedAt` опционален** (задача могла не обновляться). [source:v2x9z4rwb830hxam03jj]
- **`dueDate` и `description` опциональны** — могут отсутствовать.
- **Soft delete для `Project`**: `deletedAt: ISO8601DateTime` — nullable. `null` = активен, значение timestamp = удалён. [source:px65m8ctgvusnzczwz7c][source:rq0d3blsdg6wbljnwk56]
- **Hard delete для `Task`**: задачи удаляются физически, для них фильтр `includeDeleted` не нужен.
- **`UpdateTaskInput` — частичное обновление**: изменяемые поля опциональны; «не передал — не меняем», обязателен только `id`. [source:rq0d3blsdg6wbljnwk56]
- Внутри `Input` только скаляры и `Enum` — объектные типы запрещены. [source:rq0d3blsdg6wbljnwk56]

## SDL-контракт

```graphql
scalar ISO8601DateTime

type Project {
  id: ID!
  name: String!
  description: String
  createdAt: ISO8601DateTime!
  deletedAt: ISO8601DateTime
  tasks: [Task!]!
}

enum TaskStatus {
  BACKLOG
  TODO
  IN_PROGRESS
  DONE
  CANCELLED
}

type Task {
  id: ID!
  project: Project!
  title: String!
  description: String
  status: TaskStatus!
  dueDate: ISO8601DateTime
  createdAt: ISO8601DateTime!
  updatedAt: ISO8601DateTime
}

type TaskEdge {
  node: Task!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type TaskConnection {
  edges: [TaskEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type Query {
  getProjectById(id: ID!, includeDeleted: Boolean = false): Project
  getAllProjects(includeDeleted: Boolean = false): [Project!]!
  getTasksByStatus(projectId: ID!, status: TaskStatus!): [Task!]!
  getProjectTasks(projectId: ID!, first: Int, after: String): TaskConnection!
}

input CreateProjectInput {
  idempotenceKey: ID!
  name: String!
  description: String
}

input CreateTaskInput {
  idempotenceKey: ID!
  projectId: ID!
  title: String!
  description: String
  status: TaskStatus!
  dueDate: ISO8601DateTime
}

input UpdateTaskInput {
  id: ID!
  status: TaskStatus
  description: String
  dueDate: ISO8601DateTime
}

type Mutation {
  createProject(input: CreateProjectInput!): Project!
  createTask(input: CreateTaskInput!): Task!
  updateTask(input: UpdateTaskInput!): Task!
  deleteProject(id: ID!): Project!
  restoreProject(id: ID!): Project!
}

type Subscription {
  taskStatusChanged(projectId: ID!): Task!
}

schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}
```

---

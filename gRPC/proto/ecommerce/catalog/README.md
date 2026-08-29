# Catalog Service — каталог товаров (gRPC, Protobuf)

## Назначение

Сервис каталога товаров интернет-магазина. Предоставляет API для:

- получения списка товаров с фильтрами и пагинацией
- получения одного товара по идентификатору
- создания товара
- частичного обновления товара
- скрытия товара (soft delete)
- подписки на изменение остатков товара

Контракт описан на Protocol Buffers (proto3), генерация — через buf.

## Структура

```
gRPC/
├── buf.yaml
├── buf.gen.yaml
├── buf.lock
├── proto/
│   └── ecommerce/
│       ├── common/v1/
│       │   └── money.proto
│       └── catalog/v1/
│           ├── category.proto
│           ├── product_status.proto
│           ├── product.proto
│           ├── product_summary.proto
│           ├── list_products_request.proto
│           ├── list_products_response.proto
│           ├── get_product_request.proto
│           ├── get_product_response.proto
│           ├── create_product_request.proto
│           ├── create_product_response.proto
│           ├── update_product_request.proto
│           ├── update_product_response.proto
│           ├── hide_product_request.proto
│           ├── hide_product_response.proto
│           ├── watch_stock_request.proto
│           ├── stock_event.proto
│           └── catalog_service.proto
└── gen/
    ├── ecommerce/
    │   ├── common/v1/money.pb.go
    │   └── catalog/v1/
    │       ├── *.pb.go
    │       └── catalog_service_grpc.pb.go
    ├── gateway/ecommerce/catalog/v1/catalog_service.pb.gw.go
    └── openapiv2/ecommerce/catalog/v1/catalog_service.swagger.json
```

## Модель данных

### Перечисления

```proto
enum ProductStatus {
  PRODUCT_STATUS_UNSPECIFIED = 0;
  PRODUCT_STATUS_ACTIVE = 1;
  PRODUCT_STATUS_HIDDEN = 2;
  PRODUCT_STATUS_ARCHIVED = 3;
}
```

### Сообщения

|
 Тип 
|
 Назначение 
|
|
---
|
---
|
|
`Category`
|
 категория, вложенная через 
`parent_id`
|
|
`Product`
|
 полная сущность товара 
|
|
`ProductSummary`
|
 лёгкая карточка для списка, без 
`description`
|
|
`Money`
|
 общий тип денег из 
`ecommerce.common.v1`
|
|
`StockEvent`
|
 событие изменения остатка 
|

## Методы API

|
 Метод 
|
 RPC-стиль 
|
 Назначение 
|
|
---
|
---
|
---
|
|
`ListProducts`
|
 unary 
|
 список с фильтрами и пагинацией 
|
|
`GetProduct`
|
 unary 
|
 один товар по id 
|
|
`CreateProduct`
|
 unary 
|
 создать товар 
|
|
`UpdateProduct`
|
 unary 
|
 частичное обновление через FieldMask 
|
|
`HideProduct`
|
 unary 
|
 скрыть товар (soft delete) 
|
|
`WatchStock`
|
 server-streaming 
|
 подписка на изменения остатков 
|

## Ключевые решения

- **Деньги** — `ecommerce.common.v1.Money`; сумма в минорных единицах + код валюты ISO 4217, никогда `double`.
- **Enum** начинается с `*_UNSPECIFIED = 0`; значения префиксованы именем enum.
- **Категория** — `message`, а не `enum`: категории образуют дерево и управляются данными, enum умеет только плоский закрытый список.
- **Пагинация** — `page_size` + `page_token`/`next_page_token`, а не `offset`/`limit`. Токен — непрозрачная для клиента строка.
- **Частичное обновление** — `google.protobuf.FieldMask`: сервер меняет только перечисленные поля.
- **Мягкое удаление** — отдельный метод `HideProduct`: клиент не передаёт произвольный статус, сервер сам переводит в `PRODUCT_STATUS_HIDDEN`.
- **`ProductSummary`** — в списке возвращаем карточку без описания, чтобы не раздувать ответ.
- **`CreateProduct`** с обязательным `idempotency_key` — защита от дублей при повторных запросах.
- **Подписка на остатки** — server-streaming: один запрос, поток `StockEvent`.

## HTTP-аннотации

В `catalog_service.proto` для каждого RPC добавлены аннотации `google.api.http`, по которым buf генерирует gRPC Gateway и OpenAPI-документацию.

## Генерация

```bash
buf dep update
buf lint
rm -rf gen
buf generate
```

Артефакты:

- `gen/ecommerce/catalog/v1/*.pb.go` — структуры
- `gen/ecommerce/catalog/v1/catalog_service_grpc.pb.go` — gRPC-стабы
- `gen/gateway/ecommerce/catalog/v1/catalog_service.pb.gw.go` — HTTP-гейтвей
- `gen/openapiv2/ecommerce/catalog/v1/catalog_service.swagger.json` — OpenAPI

## Просмотр OpenAPI

```bash
swagger serve gen/openapiv2/ecommerce/catalog/v1/catalog_service.swagger.json -p 8082
```

## Обратная совместимость

- Безопасно: добавить поле, добавить enum-значение, зарезервировать удалённое поле.
- Ломающее: сменить номер поля, перенести поле в `oneof`.
- Ломающие изменения — новая версия пакета `v2`.
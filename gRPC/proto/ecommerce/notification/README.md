# Notification Service — сервис уведомлений (gRPC, Protobuf)

## Назначение

Сервис отправки и доставки уведомлений пользователям. Демонстрирует **все четыре типа RPC** в gRPC:

- unary — отправка одного уведомления;
- client-streaming — массовая рассылка уведомлений;
- server-streaming — подписка на входящие уведомления;
- bidirectional — двусторонняя синхронизация статусов уведомлений.

Контракт описан на Protocol Buffers (proto3), генерация — через buf.

## Структура

```
gRPC/
├── buf.yaml
├── buf.gen.yaml
├── buf.lock
├── proto/
│   └── ecommerce/notification/v1/
│       ├── delivery_channel.proto
│       ├── delivery_status.proto
│       ├── notification_priority.proto
│       ├── notification_template.proto
│       ├── notification.proto
│       ├── notification_event.proto
│       ├── notification_status_update.proto
│       ├── send_notification_request.proto
│       ├── send_notification_response.proto
│       ├── bulk_send_notification_input.proto
│       ├── bulk_send_failure.proto
│       ├── bulk_send_notifications_response.proto
│       ├── subscribe_notifications_request.proto
│       └── notification_service.proto
└── gen/
    └── ecommerce/notification/v1/
        ├── *.pb.go
        └── notification_service_grpc.pb.go
```

Каждый `.proto`-файл содержит ровно один top-level элемент — правило **1-1-1**.

## Модель данных

### Перечисления

```proto
enum DeliveryChannel {
  DELIVERY_CHANNEL_UNSPECIFIED = 0;
  DELIVERY_CHANNEL_EMAIL = 1;
  DELIVERY_CHANNEL_SMS = 2;
  DELIVERY_CHANNEL_PUSH = 3;
}

enum DeliveryStatus {
  DELIVERY_STATUS_UNSPECIFIED = 0;
  DELIVERY_STATUS_PENDING = 1;
  DELIVERY_STATUS_SENT = 2;
  DELIVERY_STATUS_DELIVERED = 3;
  DELIVERY_STATUS_FAILED = 4;
}

enum NotificationPriority {
  NOTIFICATION_PRIORITY_UNSPECIFIED = 0;
  NOTIFICATION_PRIORITY_LOW = 1;
  NOTIFICATION_PRIORITY_NORMAL = 2;
  NOTIFICATION_PRIORITY_HIGH = 3;
}
```

Первое значение всегда `*_UNSPECIFIED = 0` — это значение по умолчанию, которое означает «не задано», а не реальное значение.

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
`Notification`
|
 готовое уведомление для доставки 
|
|
`NotificationTemplate`
|
 шаблон с плейсхолдерами 
|
|
`NotificationEvent`
|
 событие для потоков от сервера 
|
|
`NotificationStatusUpdate`
|
 сообщение от клиента в потоке синхронизации 
|
|
`BulkSendNotificationInput`
|
 элемент потока массовой рассылки 
|
|
`BulkSendFailure`
|
 информация о неудачной отправке 
|
|
`BulkSendNotificationsResponse`
|
 итоговый ответ массовой рассылки 
|

## Методы API

`SendNotification`
|
 unary 
|
 отправить одно уведомление 
|
|
`BulkSendNotifications`
|
 client-streaming 
|
 массовая рассылка 
|
|
`SubscribeNotifications`
|
 server-streaming 
|
 подписка на входящие 
|
|
`SyncNotificationStatus`
|
 bidirectional 
|
 синхронизация статусов 
|

### Почему `BulkSendNotifications` — client-streaming

Поток идёт **от клиента к серверу** [1](#ref-source-u9y15e8r3d9d69x2ytt1). Клиент отправляет уведомления частями, не собирая тысячу элементов в один гигантский `repeated`. Это экономит память, не упирается в лимит размера сообщения и позволяет серверу обрабатывать данные по мере поступления.

### Почему `SyncNotificationStatus` — bidirectional

Обе стороны независимо шлют потоки:

- клиент → сервер: `NotificationStatusUpdate` («уведомление прочитано», «доставлено»);
- сервер → клиент: `NotificationEvent` (новое уведомление, подтверждение статуса).

Это живой двусторонний канал синхронизации.

## Ключевые решения

- **Каналы, статусы и приоритет** — enum, потому что это закрытые наборы значений.
- **Пользователь** — только `user_id`, а не вложенный `User`: пользователи управляются отдельным сервисом.
- **`NotificationTemplate`** — отдельная сущность, на которую ссылается `Notification` по `template_id`.
- **`optional`** для `template_id`, `scheduled_at`, `sent_at` — эти поля могут отсутствовать.
- **`idempotency` / `batch_id`** — ключ рассылки, передаваемый в каждом элементе потока, защищает от дублей при повторной массовой рассылке.

## Почему сервис без HTTP-аннотаций

Client-streaming и bidirectional RPC нельзя корректно представить обычным REST-запросом/ответом. Поэтому `notification_service.proto` описан как чистый gRPC-контракт без `google.api.http`-аннотаций [1](#ref-source-u9y15e8r3d9d69x2ytt1).

При желании REST-слой для методов `SendNotification` и `SubscribeNotifications` можно сгенерировать отдельно, не смешивая его с bidirectional-методами.

## Генерация

```bash
buf dep update
buf lint
rm -rf gen
buf generate
```

Артефакты:

- `gen/ecommerce/notification/v1/*.pb.go` — структуры;
- `gen/ecommerce/notification/v1/notification_service_grpc.pb.go` — gRPC-стабы со всеми четырьмя стилями RPC.

## Обратная совместимость

- Безопасно: добавить поле, добавить enum-значение, зарезервировать удалённое поле.
- Ломающее: сменить номер поля, перенести поле в `oneof`.
- Ломающие изменения — новая версия пакета `v2`.
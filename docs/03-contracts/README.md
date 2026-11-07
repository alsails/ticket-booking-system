# 03. Контракты OpenAPI / AsyncAPI

**Проект:** Микросервисная платформа покупки билетов независимых организаторов с распределённым бронированием и компенсацией частично выполненных заказов.

## 1. Состав файлов

| Файл | Стандарт | Кто предоставляет → кто вызывает |
|---|---|---|
| [`order-api.openapi.yaml`](order-api.openapi.yaml) | OpenAPI 3.0.3 | Order Service → Web UI / HTTP-клиент |
| [`organizer-api.openapi.yaml`](organizer-api.openapi.yaml) | OpenAPI 3.0.3 | **Каждый** Organizer Booking Service (1..N) → Order Service |
| [`payment-api.openapi.yaml`](payment-api.openapi.yaml) | OpenAPI 3.0.3 | Payment Service → Order Service; входящий webhook провайдера → Payment Service |
| [`provider-api.openapi.yaml`](provider-api.openapi.yaml) | OpenAPI 3.0.3 | Внешний платёжный провайдер (в MVP — имитатор) → Payment Service |
| [`platform-events.asyncapi.yaml`](platform-events.asyncapi.yaml) | AsyncAPI 3.0.0 | Payment Service → RabbitMQ → Order Service |

`organizer-api.openapi.yaml` применяется **одинаково ко всем организаторам**, независимо от их количества. 

## 2. Список HTTP-операций

| Система | Метод и маршрут | Назначение |
|---|---|---|
| Order | `GET /organizers` | Список подключённых организаторов |
| Order | `GET /organizers/{organizerId}/events` | Справочный каталог одного организатора |
| Order | `GET /organizers/{organizerId}/events/{eventId}` | Свободные места и справочная цена |
| Order | `POST /orders` | Принять составной заказ, запустить Saga |
| Order | `GET /orders/{orderId}` | Состояние заказа, выдача всего комплекта билетов **только** в `COMPLETED` |
| Organizer | `GET /events` / `GET /events/{eventId}` | Свои мероприятия и места |
| Organizer | `POST /reservations` | **Атомарно** удержать несколько мест у одного организатора |
| Organizer | `GET /reservations/{reservationId}` | Проверить бронь и список билетов |
| Organizer | `POST /reservations/{reservationId}/confirm` | Подтвердить бронь и выпустить билеты |
| Organizer | `POST /reservations/{reservationId}/cancel` | Отменить временную бронь |
| Organizer | `POST /tickets/{ticketId}/revoke` | Аннулировать подтверждённый неиспользованный билет |
| Organizer | `GET /operations/{operationId}` | Сверить исход Reserve/Confirm/Cancel/Revoke после таймаута |
| Payment | `POST /payments` | Начать оплату, без обещания синхронного финала |
| Payment | `GET /payments/{paymentId}` | Получить статус оплаты |
| Payment | `GET /payments/by-operation/{operationId}` | Найти оплату после потери ответа |
| Payment | `POST /payments/{paymentId}/refunds` | Начать **полный** возврат |
| Payment | `GET /refunds/by-operation/{operationId}` | Сверить возврат по исходной операции |
| Payment | `POST /provider-webhooks/transactions` | Принять подписанное уведомление провайдера |
| Provider | `POST /charges`, `GET /charges/by-operation/{operationId}` | Списание и сверка |
| Provider | `POST /charges/{chargeId}/refunds`, `GET /refunds/by-operation/{operationId}` | Возврат и сверка |

Пути приведены относительно базового URL из соответствующего YAML (`/api/v1`). 

## 3. Договорённости о данных

1. **Произвольное количество позиций.** `POST /orders` принимает массив `items` длиной от 1; одна позиция описывает `organizer_id`, `event_id` и набор `seat_ids`. Разрешены разные мероприятия одного организатора и разные организаторы. Запрещены дубли одного `(organizer_id, event_id, seat_id)` в заказе.
2. **Места с конкретными номерами.** Работает с назначенными местами (`seat_ids`), а не с безместными квотами. 
3. **Суммы без `float`.** `amount_minor`, `price_minor`, `total_minor` — целые числа в копейках; единственная валюта `RUB`. Клиент не передаёт цену; итог рассчитывается по подтверждённым резервам организаторов.
4. **Источники истины.** `organizer_id + event_id + seat_id` уникально у организатора. Справочная доступность `GET /events` не означает блокировку. Только успешный `POST /reservations` удерживает место, включая продажи через собственные каналы организатора.
5. **Срок брони.** `requested_hold_seconds` — минимум, гарантируемый организатором. Используем 900 секунд; система отказывает `HOLD_UNSUPPORTED`, если гарантировать этот срок не может. Order Service проверяет оставшееся время до запуска платежа; при истечении до Confirm возможна компенсация.
6. **Один резерв на организатора.** Order Service группирует **все позиции одного заказа одного организатора** в один `POST /reservations`. Операция организатора атомарна для всех переданных мест. С несколькими организаторами глобальной атомарности нет.
7. **Билеты после завершения.** Организатор создаёт билеты в `CONFIRMED`, но платформа **не возвращает идентификаторы билетов покупателю**, пока весь заказ не достиг `COMPLETED`. Дальнейшая интеграция с контролем прохода за пределами MVP.
8. **Покупка целиком.** Если хотя бы один шаг невозможно завершить, заказ уходит в `COMPENSATING`; ранее полученные временные брони отменяются, подтверждённые билеты аннулируются, платёж возвращается. `CANCELLED` устанавливается только после подтверждения нужных компенсаций.

## 4. Идемпотентность и сетевые ошибки

**Все командные изменяющие операции** (кроме входящего webhook) принимают неизменный `Idempotency-Key` (UUID). Webhook дедуплицируется отдельно по `provider_event_id`. В межсервисных командах также есть стабильный `operation_id` (UUID), связанный с конкретным шагом Saga. При повторе **той же** команды с теми же параметрами возвращается её прежний результат, не выполняется новое списание/бронь/возврат. Тот же ключ с другими параметрами возвращает `409`.

Схематичный пример: `Order Service` создаёт команду `RESERVE` с `operation_id=R1` для организатора №3; HTTP-ответ теряется. `Order Service` запрашивает `GET /operations/R1`; если ответ всё ещё неизвестен, безопасно повторяет Reserve **с тем же** `operation_id` и заголовком `Idempotency-Key`, а не создаёт новую бронь.

Для денежной операции `Payment Service` действует аналогично: локально сохраняет `operation_id`, вызывает провайдера с устойчивым ключом, а при потере ответа обращается к `GET /charges/by-operation/{operationId}` либо повторяет исходный `POST /charges` с тем же ключом. Уведомления webhook проверяются криптографически и дедуплицируются по `provider_event_id`. **HTTP 503 и таймаут не означают, что платёж не состоялся.**

Дополнительные гарантии реализации: уникальные индексы на ключах идемпотентности, хранение отпечатка запроса и результата, транзакционная запись шагов Saga, периодическая сверка зависших операций, ограниченные сетевые retry и последующая фоновая обработка. 

## 5. События RabbitMQ

Exchange: **`ticket.payment.events`**, тип `topic`, долговечный. Очередь Order Service: **`order.saga.payment-events`**, долговечная, привязана к следующим routing keys:

| Routing key | Сообщение | Что делает Order Service |
|---|---|---|
| `payment.succeeded.v1` | `PaymentSucceeded` | Сохраняет факт успешной оплаты, подтверждает резервы |
| `payment.failed.v1` | `PaymentFailed` | При **окончательном** отказе оплаты отменяет временные брони |
| `refund.succeeded.v1` | `RefundSucceeded` | Подтверждает денежный шаг компенсации |
| `refund.failed.v1` | `RefundFailed` | Не завершает заказ отменой, сверяет/эскалирует возврат |

**Гарантия: at-least-once, не exactly-once.** Payment Service записывает изменение платежа и событие в **одну локальную транзакцию** (transactional outbox), затем фоновый publisher отправляет сообщение с publisher confirms в RabbitMQ. Order Service выполняет обработку и запись `event_id` в inbox в одной локальной транзакции, после чего подтверждает сообщение RabbitMQ. Дубли игнорируются, сообщения в неверном для Saga состоянии не запускают повторные внешние действия. Ошибочные сообщения попадают в DLQ / подлежат разбору; события не должны удаляться без учёта.

В AsyncAPI 3 у операций направления обозначены явно: `send` — отправка Payment Service, `receive` — получение Order Service. Внешние организаторы используют REST и **не обязаны подключаться** к брокеру.

### Пример события `PaymentSucceeded`

```json
{
  "event_id": "8979014b-98c6-45f7-81e4-838edbbfb157",
  "event_type": "PaymentSucceeded",
  "event_version": 1,
  "occurred_at": "2026-11-07T19:00:00Z",
  "correlation_id": "5dfc83b6-3859-4aa9-aa65-f868c7d2ca0f",
  "order_id": "5dfc83b6-3859-4aa9-aa65-f868c7d2ca0f",
  "data": {
    "payment_id": "978da555-d640-47c2-8021-dcfe46ac0914",
    "payment_operation_id": "59708052-80cb-4b94-a4ee-32408c3972dc",
    "provider_charge_id": "ch_001",
    "amount_minor": 250000,
    "currency": "RUB"
  }
}
```

## 6. Соответствие сценариям сбоя

| Сценарий задания | Контрактные механизмы |
|---|---|
| Организатор A забронировал, B отказал | `Reserve` B → `409 SEAT_UNAVAILABLE`; `Cancel` A; Order Service завершает `CANCELLED` после подтверждения |
| Провайдер списал деньги, но ответ потерялся | `GET /charges/by-operation/{operationId}`, тот же `Idempotency-Key`, проверенный webhook, `PaymentSucceeded` в RabbitMQ; повторного списания нет |
| После платежа A подтвердил, B окончательно отказал | `Revoke` всех подтверждённых билетов A, `Cancel` оставшихся HELD, `POST /payments/{id}/refunds`, ожидание `RefundSucceeded`; до конца `COMPENSATING` |

Если какой-то сервис недоступен, промежуточное состояние сохраняется, фоновые повторы и сверка продолжаются. При невозможности автоматического урегулирования статус `MANUAL_REVIEW` сохраняет обязательства; это **не** ложный успех отмены.
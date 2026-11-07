# 04.1. Успешный сценарий: покупка всех билетов заказа

**Предусловия:** покупатель выбирает одно или несколько мероприятий у одного или нескольких независимых организаторов. Все организаторы реализуют общий API; записи о площадках и местах хранятся **только у них**. Для каждого уникального `organizer_id` Order Service делает один атомарный резерв всего набора мест этого организатора.

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as Покупатель
    participant Order as Order Service (Saga)
    participant ODB as Order DB
    participant Org as Organizer API (1..N)
    participant Pay as Payment Service
    participant PDB as Payment DB
    participant Provider as Payment Provider
    participant MQ as RabbitMQ

    Buyer->>Order: POST /orders (items[1..N], Idempotency-Key)
    Order->>ODB: Сохранить заказ CREATED и шаги Saga
    Order-->>Buyer: 202 Accepted + order_id
    Order->>ODB: Статус RESERVING
    loop Для каждого уникального organizer_id
        Order->>ODB: Зафиксировать шаг RESERVE с OP_R и K_R
        Order->>Org: POST /reservations (operation_id=OP_R, items[], hold=900)
        Org-->>Order: 201 HELD + reservation_id + expires_at + total_minor
        Order->>ODB: Сохранить бронь, сумму, срок и результат
    end
    Order->>Order: Проверить все HELD, expires_at и RUB; суммировать цены
    Order->>ODB: Статус PAYMENT_PENDING, шаг CHARGE (OP_P, K_P)
    Order->>Pay: POST /payments (OP_P, order_id, amount_minor, RUB)
    Pay->>PDB: Сохранить платёж PENDING и ключ операции
    Pay-->>Order: 202 PENDING + payment_id
    Pay->>Provider: POST /charges (OP_P, Idempotency-Key=K_P)
    Provider-->>Pay: 202 PENDING
    Provider->>Pay: POST /provider-webhooks/transactions (signed SUCCEEDED)
    Pay->>Provider: GET /charges/by-operation/OP_P
    Provider-->>Pay: SUCCEEDED + provider_charge_id
    Pay->>PDB: Одна транзакция - SUCCEEDED + outbox PaymentSucceeded
    Pay->>MQ: Publish PaymentSucceeded (outbox + confirms)
    MQ-->>Order: Deliver PaymentSucceeded (event_id, order_id)
    Order->>ODB: Inbox по event_id + PAYMENT_PENDING -> CONFIRMING
    Order-->>MQ: ACK после фиксации
    loop Для каждого сохранённого HELD reservation_id
        Order->>ODB: Сохранить шаг CONFIRM (OP_C, K_C)
        Order->>Org: POST /reservations/{reservationId}/confirm (OP_C)
        Org-->>Order: 200 CONFIRMED + tickets ISSUED[]
        Order->>ODB: Сохранить каждый ticket_id и подтверждение
    end
    Order->>ODB: Атомарно установить COMPLETED, когда все билеты подтверждены
    Buyer->>Order: GET /orders/{orderId}
    Order->>ODB: Прочитать итог и весь комплект
    Order-->>Buyer: 200 COMPLETED + confirmed_ticket_id для всех мест
```

## Что гарантирует этот сценарий

- `POST /orders` возвращает `202`, поскольку оформление выполняется **асинхронно**; пользователь проверяет готовность через `GET /orders/{orderId}`.
- Организатор остаётся владельцем места и билета. `GET /events` не резервирует место; только локально атомарный `POST /reservations` учитывает конкурирующие продажи.
- `PaymentSucceeded` возникает после **верификации** результата у провайдера. Изменение состояния Payment DB и outbox-событие записываются одной локальной транзакцией; порядок публикации и обработки восстанавливаем.
- Платёжная сумма вычисляется по `total_minor` всех успешных броней, а не по цене, отправленной клиентом; учебная валюта — `RUB`.
- Даже если организатор уже выпустил билеты, покупатель не получит их номера, пока **все** они не подтверждены и заказ не переведён в `COMPLETED`.
- Повтор `PaymentSucceeded` не создаёт повторную Saga: Order Service дедуплицирует сообщения через inbox, а шаги Saga записывает в Order DB.

**Постусловие:** `Order.status=COMPLETED`, все заказанные `(organizer_id, event_id, seat_id)` имеют `confirmed_ticket_id`, существует ровно одно проверенное успешное списание общей суммы.

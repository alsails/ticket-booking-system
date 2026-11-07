# 04.3. Сбой №2: деньги списаны, ответ платёжного провайдера потерян

**Пример:** все нужные брони уже находятся в `HELD`. Payment Service направил единственную команду списания, провайдер её выполнил, но HTTP-ответ не дошёл обратно. **Timeout ≠ PaymentFailed**: мы не отправляем новое списание с другим ключом и не считаем деньги несписанными.

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

    Note over Order,Org: Все брони HELD; Order.status=PAYMENT_PENDING
    Order->>ODB: Сохранён шаг CHARGE (OP_P, K_P), ожидаем результат
    Order->>Pay: POST /payments (OP_P, Idempotency-Key=K_P)
    Pay->>PDB: Зафиксировать единственный платёж PENDING
    Pay-->>Order: 202 PENDING + payment_id
    Pay->>Provider: POST /charges (OP_P, Idempotency-Key=K_P)
    Provider->>Provider: Списать сумму ОДИН раз, записать SUCCEEDED
    Provider--xPay: HTTP-ответ потерян (timeout)
    Pay->>PDB: Сохранить PENDING, исход неизвестен
    Note over Pay,Provider: Никакого второго списания с новым ключом
    Pay->>Provider: GET /charges/by-operation/OP_P
    alt Провайдер отвечает и подтверждает списание
        Provider-->>Pay: 200 SUCCEEDED + provider_charge_id
    else Сверка временно недоступна
        Provider--xPay: 503 / timeout
        Pay->>PDB: Оставить PENDING и запланировать повторную сверку
        Provider->>Pay: POST /provider-webhooks/transactions (signed SUCCEEDED)
        Pay->>Provider: GET /charges/by-operation/OP_P (повтор сверки)
        Provider-->>Pay: 200 SUCCEEDED + provider_charge_id
    end
    Pay->>PDB: Одной транзакцией SUCCEEDED + outbox PaymentSucceeded
    Pay->>MQ: Publish PaymentSucceeded (publisher confirms)
    MQ-->>Order: PaymentSucceeded (event_id, order_id, OP_P)
    Order->>ODB: Inbox + подтверждение платежа, переход в CONFIRMING
    Order-->>MQ: ACK
    alt Все временные брони всё ещё действительны
        loop Для каждого organiser_id (1..N)
            Order->>Org: POST /reservations/{reservationId}/confirm (OP_C, K_C)
            Org-->>Order: 200 CONFIRMED + билеты
            Order->>ODB: Сохранить результаты и ticket_id
        end
        Order->>ODB: COMPLETED после подтверждения всех билетов
    else Пока выясняли платёж, бронь истекла
        Order->>ODB: COMPENSATING, не выдавать билеты
        Note over Order,Org: Отменить HELD, подтвердить EXPIRED или отозвать выданные билеты
        Order->>Pay: POST /payments/{paymentId}/refunds (OP_RF, K_RF)
        Note over Order,Pay: Дождаться RefundSucceeded до CANCELLED
    end
    Buyer->>Order: GET /orders/{orderId}
    Order-->>Buyer: COMPLETED / COMPENSATING / CANCELLED (по фактам)
```

## Механизм восстановления

1. Order Service один раз фиксирует `OP_P` и `K_P`, затем вызывает `POST /payments` — не напрямую внешний провайдер.
2. Payment Service записывает платёж с `operation_id=OP_P` в свою БД перед внешним вызовом. Для `POST /charges` он использует **тот же устойчивый идентификатор** списания.
3. После таймаута Payment Service вызывает `GET /charges/by-operation/{operationId}`. Допустим повтор **того же** `POST /charges` с тем же ключом и телом, если проверка по статусу не разрешает неопределённость. Запрос с новым ключом запрещён.
4. При подтверждении списания сохраняется `SUCCEEDED` и исходящее событие `PaymentSucceeded` в одной локальной транзакции. RabbitMQ доставляет сообщение минимум один раз; Order Service применяет его один раз по `event_id`.
5. Если брони всё ещё активны, Saga продолжает подтверждение билетов. Если бронь уже истекла, Saga начинает полноценную компенсацию (включая **подтверждённый** возврат). Сам факт неизвестного платежа не даёт права считать заказ отменённым.

**Важная гонка:** если заказ уже ожидал компенсации, а `PaymentSucceeded` пришёл поздно, его нельзя игнорировать: фиксируем подтверждённое списание и инициируем/сверяем возврат. Нельзя повторно переводить такую Saga в `CONFIRMING`.

**Постусловие основной ветки:** одно списание, заказ `COMPLETED` и все билеты подтверждены. **Постусловие ветки с истекшей бронью:** один подтверждённый возврат, освобождённые ресурсы и `CANCELLED`. При недоступности провайдера/организаторов остаётся промежуточный статус, а не ложный итог.

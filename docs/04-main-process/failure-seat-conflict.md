# 04.2. Сбой №1: один организатор забронировал места, следующий отказал

**Пример:** в заказе несколько мероприятий. Организатор А удержал все свои места (`HELD`), но организатор Б вернул `409 SEAT_UNAVAILABLE` из-за продажи места через собственный канал. Отказ является **достоверным**, а не таймаутом. Для N организаторов компенсируются **все** ранее успешные брони, не только А.

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as Покупатель
    participant Order as Order Service (Saga)
    participant ODB as Order DB
    participant A as Organizer API A
    participant B as Organizer API B
    participant Pay as Payment Service

    Buyer->>Order: POST /orders (места у A и B)
    Order->>ODB: CREATED -> RESERVING
    Order-->>Buyer: 202 Accepted + order_id
    Order->>ODB: RESERVE A, сохранить OP_RA + K_RA
    Order->>A: POST /reservations (OP_RA, Idempotency-Key=K_RA)
    A-->>Order: 201 HELD + reservation_id=A_R + expires_at
    Order->>ODB: Зафиксировать HELD у A
    Order->>ODB: RESERVE B, сохранить OP_RB + K_RB
    Order->>B: POST /reservations (OP_RB, Idempotency-Key=K_RB)
    B-->>Order: 409 SEAT_UNAVAILABLE (retryable=false)
    Order->>ODB: COMPENSATING + причина SEAT_UNAVAILABLE
    Note over Order,Pay: Платёж даже не инициируется
    Order->>ODB: Сохранить шаг CANCEL A (OP_CA, K_CA)
    Order->>A: POST /reservations/A_R/cancel (OP_CA)
    alt Cancel подтверждён
        A-->>Order: 200 CANCELLED
        Order->>ODB: Зафиксировать освобождение мест A
    else Ответ потерялся или A временно недоступен
        A--xOrder: timeout / 503
        Order->>A: GET /operations/OP_CA
        A-->>Order: Текущий статус или 503
        Order->>ODB: Сохранить ожидание - статус COMPENSATING
        Note over Order,A: Worker повторяет ту же команду с OP_CA и K_CA
    end
    Order->>ODB: Если все RESERVE исходы известны и удержания сняты - CANCELLED
    Buyer->>Order: GET /orders/{orderId}
    Order-->>Buyer: CANCELLED либо COMPENSATING (пока отмена не доказана)
```

## Компенсация

1. Прервать дальнейшее бронирование и не вызывать `POST /payments`.
2. Перевести заказ в `COMPENSATING`, зафиксировать уже достигнутые результаты по каждому организатору.
3. Отменить каждую ранее подтверждённую временную бронь с помощью `POST /reservations/{reservationId}/cancel` и **стабильного** `operation_id`.
4. При неизвестном результате отмены проверить `GET /operations/{operationId}`; разрешены только идемпотентные повторы той же операции.
5. Если бронь успела истечь, подтвердить её статус `EXPIRED` через API организатора. Только после подтверждения освобождения всех ресурсов поставить `CANCELLED`.

**Инвариант:** отказ `409 SEAT_UNAVAILABLE` от одного участника **не означает** отказ от всех ранее успешных резервов автоматически — эти резервы надо компенсировать явно. Денежного возврата нет, поскольку платёж не начинался.

**Отдельный случай:** если ответ Б **не пришёл вовсе**, это не доказанный отказ. Пока исход исходной команды `OP_RB` не выяснен через `GET /operations/OP_RB` или повтор с тем же ключом, заказ нельзя считать полностью отменённым: у Б могла возникнуть активная бронь.

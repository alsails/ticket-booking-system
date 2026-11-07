# 04.4. Сбой №3: после оплаты один организатор подтвердил билеты, другой отказал

**Пример:** платёж достоверно подтверждён (`PaymentSucceeded`). Организатор А подтвердил бронь и выпустил билеты, но при обращении к Б выяснилось, что его временная бронь истекла (`409 RESERVATION_EXPIRED`). Подтверждённые билеты А **ещё не выданы покупателю**, поскольку весь заказ не завершён.

**Обязательное предварительное соглашение с каждым организатором:** по платформенным составным заказам до статуса `COMPLETED` выпущенные билеты недоступны для прохода и могут быть идемпотентно аннулированы по `POST /tickets/{ticketId}/revoke`. Если организатор этого не гарантирует, его нельзя включать в такую Saga без дополнительного механизма.

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as Покупатель
    participant Order as Order Service (Saga)
    participant ODB as Order DB
    participant A as Organizer API A
    participant B as Organizer API B
    participant Pay as Payment Service
    participant Provider as Payment Provider
    participant MQ as RabbitMQ

    Note over Order,MQ: Все резервы получены, оплата подтверждена событием PaymentSucceeded
    Order->>ODB: PAYMENT_PENDING -> CONFIRMING
    Order->>A: POST /reservations/A_R/confirm (OP_CA, K_CA)
    A-->>Order: 200 CONFIRMED + tickets [A_T1, A_T2]
    Order->>ODB: Сохранить ticket_id A_T1, A_T2; покупателю не показывать
    Order->>B: POST /reservations/B_R/confirm (OP_CB, K_CB)
    B-->>Order: 409 RESERVATION_EXPIRED (retryable=false)
    Order->>ODB: CONFIRMING -> COMPENSATING; сохранить план компенсации

    loop Для каждого уже выпущенного ticket_id у каждого организатора
        Order->>ODB: Сохранить REVOKE (OP_RV, K_RV)
        Order->>A: POST /tickets/{ticketId}/revoke (OP_RV, K_RV)
        alt Билет аннулирован
            A-->>Order: 200 REVOKED
            Order->>ODB: Зафиксировать REVOKED
        else Неизвестный результат или недоступность
            A--xOrder: timeout / 503
            Order->>A: GET /operations/OP_RV
            Order->>ODB: Оставить COMPENSATING; фоновая сверка / повтор
        end
    end
    Order->>B: GET /reservations/B_R
    B-->>Order: 200 EXPIRED (удержание прекращено)
    Order->>ODB: Зафиксировать отсутствие активной брони B
    Note over Order,Pay: Прочие оставшиеся HELD брони также Cancel + проверка
    Order->>ODB: Сохранить шаг REFUND (OP_RF, K_RF)
    Order->>Pay: POST /payments/{paymentId}/refunds (OP_RF, K_RF)
    Pay->>Provider: POST /charges/{chargeId}/refunds (OP_RF, K_RF)
    Provider-->>Pay: 202 PENDING
    Provider->>Pay: POST /provider-webhooks/transactions (signed REFUND SUCCEEDED)
    Pay->>Provider: GET /refunds/by-operation/OP_RF
    Provider-->>Pay: 200 SUCCEEDED (полный возврат)
    Pay->>MQ: Publish RefundSucceeded через transactional outbox
    MQ-->>Order: RefundSucceeded (event_id, order_id, OP_RF)
    Order->>ODB: Inbox + факт успешного полного возврата
    Order-->>MQ: ACK
    alt Все билеты REVOKED, удержания сняты, RefundSucceeded
        Order->>ODB: COMPENSATING -> CANCELLED
        Buyer->>Order: GET /orders/{orderId}
        Order-->>Buyer: CANCELLED (денег нет, действующих билетов нет)
    else Компенсация ещё не подтверждена
        Order->>ODB: COMPENSATING или MANUAL_REVIEW
        Buyer->>Order: GET /orders/{orderId}
        Order-->>Buyer: COMPENSATING / MANUAL_REVIEW (не CANCELLED)
    end
```

## Как компенсируем частичный успех

1. Сохраняем отказ Б и состояние `COMPENSATING`; **не** переводим заказ в `COMPLETED` и не показываем покупателю `confirmed_ticket_id`.
2. Для каждого ранее подтверждённого билета (`ISSUED`) вызываем отдельный `POST /tickets/{ticketId}/revoke`. При нескольких билетах А их все надо аннулировать; аналогично и другим организаторам, если они успели подтвердить билеты.
3. Для остальных удержаний в `HELD` выполняем `POST /reservations/{reservationId}/cancel`, для уже истёкших проверяем `EXPIRED`. При неопределённом результате `Confirm` перед компенсацией выясняем его через `GET /operations/{operationId}`: подтверждённый билет нужно именно аннулировать, а не пытаться отменить `HELD`.
4. Создаём **одну** операцию полного возврата `POST /payments/{paymentId}/refunds` со стабильными `OP_RF` и `K_RF`. Payment Service проверяет у провайдера конечный результат; событие `RefundSucceeded` означает действительно выполненный возврат.
5. После подтверждения всех аннулирований, освобождений и возврата Order Service устанавливает `CANCELLED`. Если операция недоступна — сохраняет `COMPENSATING` и повторяет; при невозможности автоматически устранить конфликт — `MANUAL_REVIEW`.

## Если компенсация тоже сломалась

| Проблема | Реакция |
|---|---|
| `Revoke` вернул `503` или timeout | Проверить `GET /operations/OP_RV`; повторять с тем же ключом, не выдавая билет пользователю |
| Билет оказался использован (`409 TICKET_ALREADY_USED`) | Не объявлять заказ отменённым; зафиксировать `MANUAL_REVIEW` и ручное урегулирование |
| Возврат `PENDING`, webhook ещё нет | Оставить `COMPENSATING`, сверять `GET /refunds/by-operation/OP_RF`, ожидать `RefundSucceeded` |
| Событие `RefundSucceeded` доставлено дважды | Дедупликация по `event_id` в Order DB, второй раз компенсация не запускается |
| `RefundFailed` подтверждён | Сохранить незавершённый результат, провести сверку и эскалацию; нельзя ставить `CANCELLED` |

**Постусловие нормальной компенсации:** все билеты из заказа недействительны; временные резервы отменены или истекли; полная сумма возвращена; `Order.status=CANCELLED`. **Постусловие неразрешённой ситуации:** `COMPENSATING` либо `MANUAL_REVIEW`, полный журнал действий и отсутствие ложного сообщения об отмене.

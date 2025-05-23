# Справочник API для Push-модели Купер API

## Базовые endpoint'ы

| Назначение | URL | Метод |
|------------|-----|--------|
| Уведомления от мерчанта | `https://merchant-api.sbermarket.ru/ofm/api/v1/notifications` | POST |
| Получение токена | `https://merchant-api.sbermarket.ru/auth/token` | POST |

## Вебхуки от Купера (входящие)

| Событие | Описание | Обязательный ответ | Когда приходит |
|---------|----------|-------------------|----------------|
| `order.created` | Новый заказ создан | ✅ `{status: "created", number: "ID"}` | При создании заказа |
| `order.updated` | Изменения в заказе | ❌ Статус 200 | При изменениях состава/доставки |
| `courier.assigned` | Курьер назначен | ❌ Статус 200 | После `order.in_work`, до `order.paid` |
| `courier.arrived` | Курьер прибыл | ❌ Статус 200 | После `courier.assigned`, до `order.paid` |
| `order.paid` | Заказ оплачен | ❌ Статус 200 | После `order.ready_for_delivery` |
| `order.received` | Курьер забрал заказ | ❌ Статус 200 | После `order.paid` |
| `order.delivering` | Заказ доставляется | ❌ Статус 200 | После `order.received` |
| `order.delivered` | Заказ доставлен | ❌ Статус 200 | При завершении доставки |
| `order.canceled` | Заказ отменен | ❌ Статус 200 | При отмене |

## Уведомления от мерчанта (исходящие)

### Общие для всех типов интеграции:

| Уведомление | Описание | Когда отправлять | Обязательно |
|-------------|----------|------------------|-------------|
| `order.accepted` | Заказ принят | Для ресторанов после `order.created` | ✅ Для ресторанов |
| `order.in_work` | Сборка началась | При начале сборки | ✅ |
| `order.assembled` | Заказ собран | После завершения сборки | ✅ |
| `order.ready_for_delivery` | Готов к выдаче | После сборки + итоговый состав | ✅ |
| `order.canceled` | Отмена заказа | При отмене со стороны мерчанта | При необходимости |

### Специфичные для типов интеграции:

| Уведомление | Тип интеграции | Описание | Когда отправлять |
|-------------|----------------|----------|------------------|
| `order.pickup_code_created` | Самовывоз | Код выдачи создан | После `order.ready_for_delivery` |
| `order.estimated_delivery_time` | Доставка мерчанта | Время доставки | После `order.paid` |
| `order.delivering` | Доставка мерчанта | Начало доставки | При отправке курьера |
| `order.delivered` | Самовывоз/Доставка мерчанта | Заказ выдан | При выдаче заказа клиенту |

## Примеры запросов

### 1. Ответ на order.created

```json
POST /webhook-endpoint
Content-Type: application/json
Authorization: Bearer your-token

Response:
{
  "status": "created",
  "number": "MERCHANT_ORDER_12345",
  "expectedAssemblyTime": "2024-01-15T14:30:00+03:00"
}
```

### 2. Уведомление order.in_work

```json
POST https://merchant-api.sbermarket.ru/ofm/api/v1/notifications
Content-Type: application/json
Authorization: Bearer your-token

{
  "event_type": "order.in_work",
  "payload": {
    "orderUUID": "retailer-slug:265cb601-a78a-4862-9e9d-d6b48d6a0a3f",
    "number": "MERCHANT_ORDER_12345"
  }
}
```

### 3. Уведомление order.ready_for_delivery

```json
POST https://merchant-api.sbermarket.ru/ofm/api/v1/notifications
Content-Type: application/json
Authorization: Bearer your-token

{
  "event_type": "order.ready_for_delivery",
  "payload": {
    "orderUUID": "retailer-slug:265cb601-a78a-4862-9e9d-d6b48d6a0a3f",
    "number": "MERCHANT_ORDER_12345",
    "positions": [
      {
        "id": "123456",
        "quantity": 1,
        "price": "199.99",
        "discountPrice": "189.99",
        "totalPrice": "189.99",
        "weight": "500"
      }
    ],
    "total": {
      "totalPrice": "189.99",
      "discountTotalPrice": "189.99"
    }
  }
}
```

### 4. Уведомление order.pickup_code_created (для самовывоза)

```json
POST https://merchant-api.sbermarket.ru/ofm/api/v1/notifications
Content-Type: application/json
Authorization: Bearer your-token

{
  "event_type": "order.pickup_code_created",
  "payload": {
    "orderUUID": "retailer-slug:265cb601-a78a-4862-9e9d-d6b48d6a0a3f",
    "number": "MERCHANT_ORDER_12345",
    "pickupCode": "1234"
  }
}
```

### 5. Уведомление order.delivered (для самовывоза/доставки мерчанта)

```json
POST https://merchant-api.sbermarket.ru/ofm/api/v1/notifications
Content-Type: application/json
Authorization: Bearer your-token

{
  "event_type": "order.delivered",
  "payload": {
    "orderUUID": "retailer-slug:265cb601-a78a-4862-9e9d-d6b48d6a0a3f",
    "number": "MERCHANT_ORDER_12345",
    "positions": [
      {
        "id": "123456",
        "quantity": 1,
        "price": "199.99",
        "discountPrice": "189.99",
        "totalPrice": "189.99",
        "weight": "500"
      }
    ],
    "total": {
      "totalPrice": "189.99",
      "discountTotalPrice": "189.99"
    }
  }
}
```

### 6. Уведомление order.canceled

```json
POST https://merchant-api.sbermarket.ru/ofm/api/v1/notifications
Content-Type: application/json
Authorization: Bearer your-token

{
  "event_type": "order.canceled",
  "payload": {
    "orderUUID": "retailer-slug:265cb601-a78a-4862-9e9d-d6b48d6a0a3f",
    "number": "MERCHANT_ORDER_12345",
    "cancellation": {
      "slug": "store_closed",
      "description": "Магазин закрыт на техническое обслуживание"
    }
  }
}
```

## Правильный порядок событий по типам интеграции

### Сборка мерчанта, доставка Купера:
1. `order.created` (webhook от Купера)
2. Ответ мерчанта + `order.accepted` (если ресторан)
3. `order.in_work` (уведомление от мерчанта)
4. `courier.assigned`, `courier.arrived` (опциональные webhooks от Купера)
5. `order.assembled` (уведомление от мерчанта)
6. `order.ready_for_delivery` (уведомление от мерчанта)
7. `order.paid` (webhook от Купера)
8. `order.received`, `order.delivering`, `order.delivered` (webhooks от Купера)

### Сборка мерчанта, самовывоз:
1. `order.created` (webhook от Купера)
2. Ответ мерчанта + `order.accepted` (если ресторан)
3. `order.in_work` (уведомление от мерчанта)
4. `order.assembled` (уведомление от мерчанта)
5. `order.ready_for_delivery` (уведомление от мерчанта)
6. `order.pickup_code_created` (уведомление от мерчанта)
7. `order.paid` (webhook от Купера, если оплата на стороне Купера)
8. `order.delivered` (уведомление от мерчанта при выдаче)

### Сборка мерчанта, доставка мерчанта:
1. `order.created` (webhook от Купера)
2. Ответ мерчанта + `order.accepted` (если ресторан)
3. `order.in_work` (уведомление от мерчанта)
4. `order.assembled` (уведомление от мерчанта)
5. `order.ready_for_delivery` (уведомление от мерчанта)
6. `order.paid` (webhook от Купера)
7. `order.estimated_delivery_time` (уведомление от мерчанта)
8. `order.delivering` (уведомление от мерчанта)
9. `order.delivered` (уведомление от мерчанта)

## Коды ошибок

| HTTP Status | Описание | Действие |
|-------------|----------|----------|
| 200 | Успешно | Продолжить обработку |
| 400 | Неверный запрос | Проверить формат данных |
| 401 | Ошибка авторизации | Обновить токен |
| 404 | Заказ не найден | Проверить orderUUID |
| 500 | Внутренняя ошибка | Повторить запрос позже |

## Обязательные заголовки

```http
Content-Type: application/json
Authorization: Bearer YOUR_ACCESS_TOKEN
```

## Коды причин отмены

| Slug | Описание |
|------|----------|
| `store_closed` | Магазин закрыт |
| `out_of_stock` | Товар отсутствует |
| `technical_issues` | Технические проблемы |
| `order_not_paid` | Заказ не оплачен |
| `customer_request` | Запрос клиента |

## Проверочный чек-лист для интеграции

### ✅ Обязательно реализовать:

- [ ] Endpoint для приема вебхуков от Купера
- [ ] Авторизация входящих вебхуков (Basic Auth или client-token)
- [ ] Ответ на `order.created` в правильном формате
- [ ] Отправка уведомления `order.in_work`
- [ ] Отправка уведомления `order.assembled`
- [ ] Отправка уведомления `order.ready_for_delivery` с итоговым составом
- [ ] Обработка отмен через `order.canceled`
- [ ] Логирование всех входящих и исходящих запросов

### 🔄 Специфично для типов интеграции:

#### Самовывоз:
- [ ] Отправка `order.pickup_code_created`
- [ ] Отправка `order.delivered` при выдаче заказа с итоговым составом

#### Доставка мерчанта:
- [ ] Отправка `order.estimated_delivery_time`
- [ ] Отправка `order.delivering`
- [ ] Отправка `order.delivered` при доставке с итоговым составом

### 🔄 Рекомендуется реализовать:

- [ ] Повторные попытки при ошибках API
- [ ] Мониторинг и алерты на ошибки
- [ ] Валидация входящих данных
- [ ] Обработка дубликатов вебхуков
- [ ] Timeout'ы для HTTP запросов

## Тестирование

### Сценарии для проверки:

1. **Успешный заказ (доставка Купера)**: order.created → order.in_work → courier.assigned → order.assembled → order.ready_for_delivery → order.paid → order.received → order.delivering → order.delivered

2. **Успешный заказ (самовывоз)**: order.created → order.in_work → order.assembled → order.ready_for_delivery → order.pickup_code_created → order.paid → order.delivered

3. **Успешный заказ (доставка мерчанта)**: order.created → order.in_work → order.assembled → order.ready_for_delivery → order.paid → order.estimated_delivery_time → order.delivering → order.delivered

4. **Отмена мерчантом**: order.created → order.canceled (от мерчанта)

5. **Отмена клиентом**: order.created → order.canceled (от Купера)

6. **Ресторанный заказ**: order.created → order.accepted → дальше по обычному сценарию

---

**Документация:** [docs.kuper.ru](https://docs.kuper.ru/api-products/merchant-service/orders/description) 
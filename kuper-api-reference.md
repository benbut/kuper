# Справочник API для Push-модели Купер API

## Базовые endpoint'ы

| Назначение | URL | Метод |
|------------|-----|--------|
| Уведомления от мерчанта | `https://merchant-api.sbermarket.ru/ofm/api/v1/notifications` | POST |
| Получение токена | `https://merchant-api.sbermarket.ru/auth/token` | POST |

## Вебхуки от Купера (входящие)

| Событие | Описание | Обязательный ответ |
|---------|----------|-------------------|
| `order.created` | Новый заказ создан | ✅ `{status: "created", number: "ID"}` |
| `order.updated` | Изменения в заказе | ❌ Статус 200 |
| `order.paid` | Заказ оплачен | ❌ Статус 200 |
| `order.canceled` | Заказ отменен | ❌ Статус 200 |
| `order.delivered` | Заказ доставлен | ❌ Статус 200 |
| `courier.assigned` | Курьер назначен | ❌ Статус 200 |
| `courier.arrived` | Курьер прибыл | ❌ Статус 200 |
| `order.received` | Курьер забрал заказ | ❌ Статус 200 |

## Уведомления от мерчанта (исходящие)

| Уведомление | Описание | Когда отправлять |
|-------------|----------|------------------|
| `order.accepted` | Заказ принят | Для ресторанов после `order.created` |
| `order.in_work` | Сборка началась | При начале сборки |
| `order.assembled` | Заказ собран | После завершения сборки |
| `order.ready_for_delivery` | Готов к выдаче | После сборки + итоговый состав |
| `order.canceled` | Отмена заказа | При отмене со стороны мерчанта |
| `order.pickup_code_created` | Код выдачи создан | Только для самовывоза |
| `order.delivered` | Заказ выдан | Только для самовывоза/собственной доставки |

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

### 4. Уведомление order.canceled

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

### 🔄 Рекомендуется реализовать:

- [ ] Повторные попытки при ошибках API
- [ ] Мониторинг и алерты на ошибки
- [ ] Валидация входящих данных
- [ ] Обработка дубликатов вебхуков
- [ ] Timeout'ы для HTTP запросов

## Тестирование

### Сценарии для проверки:

1. **Успешный заказ**: order.created → order.in_work → order.assembled → order.ready_for_delivery → order.paid → order.delivered
2. **Отмена мерчантом**: order.created → order.canceled (от мерчанта)
3. **Отмена клиентом**: order.created → order.canceled (от Купера)
4. **Обновление заказа**: order.created → order.updated → продолжение процесса
5. **Ресторанный заказ**: order.created → order.accepted → дальше по обычному сценарию

---

**Документация:** [docs.kuper.ru](https://docs.kuper.ru/api-products/merchant-service/orders/description) 
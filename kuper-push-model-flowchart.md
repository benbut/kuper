# Блок-схема Push-модели для работы с заказами Купер API

## Общее описание

Push-модель интеграции с Купер API предполагает, что Купер отправляет вебхуки со статусами заказов мерчанту, а мерчант отвечает на них и передает уведомления об изменениях заказа.

Основные компоненты:
- **Вебхуки** — сообщения от Купера к мерчанту
- **Уведомления** — сообщения от мерчанта к Куперу
- **Аутентификация** — через Bearer Token или Basic Auth

## Блок-схема для типа интеграции "Сборка мерчанта, доставка Купера"

```mermaid
flowchart TD
    Start([Клиент оформляет заказ]) --> A[Купер отправляет вебхук order.created]
    A --> B{Мерчант может обработать заказ?}
    
    B -->|Да| C[Мерчант отвечает status: created, number: ID заказа]
    B -->|Нет| D[Мерчант отвечает status: created, затем отправляет order.canceled]
    
    C --> E{Заказ из ресторана?}
    E -->|Да| F[Мерчант отправляет order.accepted]
    E -->|Нет| G[Мерчант начинает сборку]
    F --> G
    
    G --> H[Мерчант отправляет order.in_work]
    H --> I[Купер начинает поиск курьера]
    I --> J[Мерчант собирает заказ]
    J --> K[Мерчант отправляет order.assembled]
    K --> L[Мерчант отправляет order.ready_for_delivery с итоговым составом]
    
    L --> M[Купер списывает фактическую сумму с клиента]
    M --> N[Купер отправляет вебхук order.paid]
    N --> O[Купер может отправить courier.assigned]
    O --> P[Купер может отправить courier.arrived]
    P --> Q[Купер отправляет order.received когда курьер забирает заказ]
    Q --> R[Купер отправляет order.delivering]
    R --> S[Купер отправляет order.delivered после доставки]
    
    D --> Cancel[Заказ отменен]
    
    subgraph "Возможные отмены на любом этапе"
        AnyCancel[Любая сторона может отправить order.canceled с причиной]
    end
    
    subgraph "Дополнительные вебхуки"
        UpdateHook[order.updated - изменения в заказе]
    end
    
    S --> End([Заказ завершен])
    Cancel --> End
    
    style Start fill:#e1f5fe
    style End fill:#c8e6c9
    style Cancel fill:#ffcdd2
    style A fill:#fff3e0
    style H fill:#fff3e0
    style K fill:#fff3e0
    style L fill:#fff3e0
    style N fill:#e8f5e8
    style O fill:#e8f5e8
    style P fill:#e8f5e8
    style Q fill:#e8f5e8
    style R fill:#e8f5e8
    style S fill:#e8f5e8
```

## Детальная схема обмена сообщениями

```mermaid
sequenceDiagram
    participant Client as Клиент
    participant Kuper as Купер
    participant Merchant as Мерчант
    
    Client->>Kuper: Оформляет заказ
    Kuper->>Merchant: webhook: order.created
    
    alt Заказ можно обработать
        Merchant->>Kuper: response: {status: "created", number: "merchant_id"}
        
        opt Заказ из ресторана
            Merchant->>Kuper: notification: order.accepted
        end
        
        Merchant->>Kuper: notification: order.in_work
        Note over Kuper: Начинает поиск курьера
        
        Note over Merchant: Собирает заказ
        Merchant->>Kuper: notification: order.assembled
        
        Merchant->>Kuper: notification: order.ready_for_delivery + итоговый состав
        Note over Kuper: Списывает фактическую сумму
        
        Kuper->>Merchant: webhook: order.paid
        
        opt Курьер назначен
            Kuper->>Merchant: webhook: courier.assigned
        end
        
        opt Курьер прибыл
            Kuper->>Merchant: webhook: courier.arrived
        end
        
        Kuper->>Merchant: webhook: order.received
        Kuper->>Merchant: webhook: order.delivering
        Kuper->>Merchant: webhook: order.delivered
        
    else Заказ нельзя обработать
        Merchant->>Kuper: response: {status: "created", number: "merchant_id"}
        Merchant->>Kuper: notification: order.canceled + reason
    end
    
    opt Отмена на любом этапе
        alt Отмена со стороны Купера/клиента
            Kuper->>Merchant: webhook: order.canceled + reason
        else Отмена со стороны мерчанта
            Merchant->>Kuper: notification: order.canceled + reason
        end
    end
    
    opt Изменения в заказе
        Kuper->>Merchant: webhook: order.updated + kind
    end
```

## Ключевые моменты для реализации

### 1. Обработка вебхука order.created
- **Обязательно** ответить в формате: `{status: "created", number: "ID_в_системе_мерчанта"}`
- Если заказ нельзя обработать - ответить успешно, затем отправить `order.canceled`

### 2. Обязательные уведомления от мерчанта
- `order.accepted` - для заказов из ресторанов
- `order.in_work` - начало сборки
- `order.assembled` - заказ собран  
- `order.ready_for_delivery` - заказ готов + **итоговый состав обязателен**

### 3. Отмена заказов
- Можно отменить на любом этапе до доставки
- Всегда указывать причину отмены
- Если ошибка на этапе `order.created` - ответить успешно, затем `order.canceled`

### 4. Аутентификация
- Все запросы с Bearer Token или Basic Auth
- Поддержать авторизацию входящих вебхуков от Купера

### 5. Production endpoints
- Уведомления: `https://merchant-api.sbermarket.ru/ofm/api/v1/notifications`
- Авторизация: `https://merchant-api.sbermarket.ru/auth/token`

## Типы интеграций

### Поддерживаются 3 типа:
1. **Сборка мерчанта, доставка Купера** (показана в схеме выше)
2. **Сборка мерчанта, самовывоз** - дополнительно `order.pickup_code_created`, `order.delivered` от мерчанта
3. **Сборка мерчанта, доставка мерчанта** - дополнительно `order.estimated_delivery_time`, `order.delivering`, `order.delivered` от мерчанта

---

**Документация:** [Купер API](https://docs.kuper.ru/api-products/merchant-service/orders/description)

**Контакты:**
- Интеграция: new.partners@sbermarket.ru  
- Техподдержка: kuper-api@kuper.ru
- API вопросы: orders.api@sbermarket.ru 
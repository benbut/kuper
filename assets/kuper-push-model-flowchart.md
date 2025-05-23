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
    I --> J{Курьер найден?}
    J -->|Да| K[Купер может отправить courier.assigned]
    J -->|Нет| L[Мерчант продолжает сборку]
    K --> M[Купер может отправить courier.arrived]
    M --> L
    
    L --> N[Мерчант собирает заказ]
    N --> O[Мерчант отправляет order.assembled]
    O --> P[Мерчант отправляет order.ready_for_delivery с итоговым составом]
    
    P --> Q[Купер списывает фактическую сумму с клиента]
    Q --> R[Купер отправляет вебхук order.paid]
    R --> S[Купер отправляет order.received когда курьер забирает заказ]
    S --> T[Купер отправляет order.delivering]
    T --> U[Купер отправляет order.delivered после доставки]
    
    D --> Cancel[Заказ отменен]
    
    subgraph "Возможные отмены на любом этапе"
        AnyCancel[Любая сторона может отправить order.canceled с причиной]
    end
    
    subgraph "Дополнительные вебхуки"
        UpdateHook[order.updated - изменения в заказе]
    end
    
    U --> End([Заказ завершен])
    Cancel --> End
    
    style Start fill:#e1f5fe
    style End fill:#c8e6c9
    style Cancel fill:#ffcdd2
    style A fill:#fff3e0
    style H fill:#fff3e0
    style O fill:#fff3e0
    style P fill:#fff3e0
    style R fill:#e8f5e8
    style K fill:#e8f5e8
    style M fill:#e8f5e8
    style S fill:#e8f5e8
    style T fill:#e8f5e8
    style U fill:#e8f5e8
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
        
        opt Курьер назначен
            Kuper->>Merchant: webhook: courier.assigned
        end
        
        opt Курьер прибыл
            Kuper->>Merchant: webhook: courier.arrived
        end
        
        Note over Merchant: Собирает заказ
        Merchant->>Kuper: notification: order.assembled
        
        Merchant->>Kuper: notification: order.ready_for_delivery + итоговый состав
        Note over Kuper: Списывает фактическую сумму
        
        Kuper->>Merchant: webhook: order.paid
        
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

## Схема для типа интеграции "Сборка мерчанта, самовывоз"

```mermaid
sequenceDiagram
    participant Client as Клиент
    participant Kuper as Купер
    participant Merchant as Мерчант
    
    Client->>Kuper: Оформляет заказ
    Kuper->>Merchant: webhook: order.created
    
    Merchant->>Kuper: response: {status: "created", number: "merchant_id"}
    
    opt Заказ из ресторана
        Merchant->>Kuper: notification: order.accepted
    end
    
    Merchant->>Kuper: notification: order.in_work
    
    Note over Merchant: Собирает заказ
    Merchant->>Kuper: notification: order.assembled
    
    Merchant->>Kuper: notification: order.ready_for_delivery + итоговый состав
    
    Merchant->>Kuper: notification: order.pickup_code_created
    Note over Kuper: Показывает код выдачи клиенту
    
    opt Оплата на стороне Купера
        Note over Kuper: Списывает фактическую сумму
        Kuper->>Merchant: webhook: order.paid
    end
    
    Note over Client: Приходит за заказом
    Merchant->>Kuper: notification: order.delivered + итоговый состав
```

## Схема для типа интеграции "Сборка мерчанта, доставка мерчанта"

```mermaid
sequenceDiagram
    participant Client as Клиент
    participant Kuper as Купер
    participant Merchant as Мерчант
    
    Client->>Kuper: Оформляет заказ
    Kuper->>Merchant: webhook: order.created
    
    Merchant->>Kuper: response: {status: "created", number: "merchant_id"}
    
    opt Заказ из ресторана
        Merchant->>Kuper: notification: order.accepted
    end
    
    Merchant->>Kuper: notification: order.in_work
    
    Note over Merchant: Собирает заказ
    Merchant->>Kuper: notification: order.assembled
    
    Merchant->>Kuper: notification: order.ready_for_delivery + итоговый состав
    Note over Kuper: Списывает фактическую сумму
    
    Kuper->>Merchant: webhook: order.paid
    
    Merchant->>Kuper: notification: order.estimated_delivery_time
    Merchant->>Kuper: notification: order.delivering
    
    Note over Merchant: Доставляет заказ
    Merchant->>Kuper: notification: order.delivered + итоговый состав
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

### 3. Порядок вебхуков от Купера (тип "доставка Купера")
1. `courier.assigned`, `courier.arrived` - после `order.in_work`, до `order.paid`
2. `order.paid` - после `order.ready_for_delivery`
3. `order.received`, `order.delivering`, `order.delivered` - после `order.paid`

### 4. Дополнительные уведомления по типам интеграции
- **Самовывоз**: `order.pickup_code_created`, `order.delivered` от мерчанта
- **Доставка мерчанта**: `order.estimated_delivery_time`, `order.delivering`, `order.delivered` от мерчанта

### 5. Отмена заказов
- Можно отменить на любом этапе до доставки (до статуса `shipping`)
- Всегда указывать причину отмены
- Если ошибка на этапе `order.created` - ответить успешно, затем `order.canceled`

### 6. Аутентификация
- Все запросы с Bearer Token или Basic Auth
- Поддержать авторизацию входящих вебхуков от Купера

### 7. Production endpoints
- Уведомления: `https://merchant-api.sbermarket.ru/ofm/api/v1/notifications`
- Авторизация: `https://merchant-api.sbermarket.ru/auth/token`

## Типы интеграций

### Поддерживаются 3 типа:
1. **Сборка мерчанта, доставка Купера** - Мерчант собирает, Купер доставляет
2. **Сборка мерчанта, самовывоз** - Мерчант собирает, клиент забирает сам
3. **Сборка мерчанта, доставка мерчанта** - Мерчант собирает и доставляет сам

---

**Документация:** [Купер API](https://docs.kuper.ru/api-products/merchant-service/orders/description)

**Контакты:**
- Интеграция: new.partners@sbermarket.ru  
- Техподдержка: kuper-api@kuper.ru
- API вопросы: orders.api@sbermarket.ru 
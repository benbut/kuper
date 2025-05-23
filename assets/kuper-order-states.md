# Диаграмма состояний заказа в Push-модели Купер API

## Состояния заказа в системе Купера

```mermaid
stateDiagram-v2
    [*] --> ready : Клиент оформил заказ
    ready --> collecting : order.in_work от мерчанта
    ready --> canceled : order.canceled / timeout
    
    collecting --> assembled : order.assembled от мерчанта
    collecting --> canceled : order.canceled
    
    assembled --> ready_to_ship : order.ready_for_delivery от мерчанта + списание оплаты
    assembled --> canceled : order.canceled
    
    ready_to_ship --> shipping : Курьер забрал заказ (только для доставки Купера)
    ready_to_ship --> shipped : order.delivered от мерчанта (самовывоз/доставка мерчанта)
    ready_to_ship --> canceled : order.canceled
    
    shipping --> shipped : order.delivered от Купера (доставка Купера)
    shipping --> canceled : order.canceled (до доставки)
    
    shipped --> [*]
    canceled --> [*]
    
    note right of ready
        Статус оплаты: not_paid
        Резерв суммы на счете клиента
    end note
    
    note right of ready_to_ship
        Статус оплаты: paid
        Фактическая сумма списана
    end note
    
    note right of shipping
        Только для доставки Купера:
        order.received → order.delivering
    end note
```

## Состояния оплаты

```mermaid
stateDiagram-v2
    [*] --> not_paid : Заказ создан
    not_paid --> paid : После order.ready_for_delivery
    paid --> [*] : Заказ завершен/отменен
    
    note right of not_paid
        Сумма зарезервирована
        на счете клиента
    end note
    
    note right of paid
        Фактическая сумма списана
        после сборки заказа
    end note
```

## Детальный жизненный цикл по типам интеграции

### Для типа "Сборка мерчанта, доставка Купера"

```mermaid
timeline
    title Временная шкала событий заказа (доставка Купера)
    
    section Создание
        Клиент оформляет заказ : Заказ в Купере (статус: ready)
        order.created webhook : Уведомление мерчанту
        Response с ID мерчанта : Подтверждение от мерчанта
        
    section Подготовка  
        order.accepted (опционально) : Для ресторанов
        order.in_work notification : Начало сборки (статус: collecting)
        courier.assigned webhook : Курьер назначен (опционально)
        courier.arrived webhook : Курьер прибыл (опционально)
        
    section Сборка
        order.assembled notification : Заказ собран (статус: assembled)
        order.ready_for_delivery : Готов к выдаче + состав (статус: ready_to_ship)
        order.paid webhook : Оплата списана (оплата: paid)
        
    section Доставка
        order.received webhook : Курьер забрал заказ (статус: shipping)
        order.delivering webhook : Доставка началась
        order.delivered webhook : Заказ доставлен (статус: shipped)
```

### Для типа "Сборка мерчанта, самовывоз"

```mermaid
timeline
    title Временная шкала событий заказа (самовывоз)
    
    section Создание
        Клиент оформляет заказ : Заказ в Купере (статус: ready)
        order.created webhook : Уведомление мерчанту
        Response с ID мерчанта : Подтверждение от мерчанта
        
    section Подготовка  
        order.accepted (опционально) : Для ресторанов
        order.in_work notification : Начало сборки (статус: collecting)
        
    section Сборка
        order.assembled notification : Заказ собран (статус: assembled)
        order.ready_for_delivery : Готов к выдаче + состав (статус: ready_to_ship)
        order.pickup_code_created : Код выдачи создан
        order.paid webhook : Оплата списана (оплата: paid)
        
    section Выдача
        Клиент приходит за заказом : Клиент показывает код
        order.delivered от мерчанта : Заказ выдан + состав (статус: shipped)
```

### Для типа "Сборка мерчанта, доставка мерчанта"

```mermaid
timeline
    title Временная шкала событий заказа (доставка мерчанта)
    
    section Создание
        Клиент оформляет заказ : Заказ в Купере (статус: ready)
        order.created webhook : Уведомление мерчанту
        Response с ID мерчанта : Подтверждение от мерчанта
        
    section Подготовка  
        order.accepted (опционально) : Для ресторанов
        order.in_work notification : Начало сборки (статус: collecting)
        
    section Сборка
        order.assembled notification : Заказ собран (статус: assembled)
        order.ready_for_delivery : Готов к выдаче + состав (статус: ready_to_ship)
        order.paid webhook : Оплата списана (оплата: paid)
        
    section Доставка
        order.estimated_delivery_time : Время доставки от мерчанта
        order.delivering : Доставка началась (статус: shipping)
        order.delivered от мерчанта : Заказ доставлен + состав (статус: shipped)
```

## Обработка ошибок и отмен

```mermaid
flowchart TD
    A[Любой этап процесса] --> B{Ошибка/отмена?}
    B -->|Отмена клиентом| C[Купер отправляет order.canceled]
    B -->|Отмена Купером| D[Купер отправляет order.canceled]  
    B -->|Отмена мерчантом| E[Мерчант отправляет order.canceled]
    B -->|Технические проблемы| F[Мерчант отправляет order.canceled]
    B -->|Таймаут order.accepted| G[Купер автоматически отменяет заказ]
    
    C --> H[Статус: canceled]
    D --> H
    E --> H
    F --> H
    G --> H
    
    H --> I{Есть резерв средств?}
    I -->|Да| J[Возврат средств клиенту]
    I -->|Нет| K[Заказ завершен]
    J --> K
    
    style C fill:#ffcdd2
    style D fill:#ffcdd2  
    style E fill:#ffcdd2
    style F fill:#ffcdd2
    style G fill:#ffcdd2
    style H fill:#ffcdd2
```

## Сравнение типов интеграции

### Основные различия в статусах

```mermaid
flowchart LR
    subgraph "Доставка Купера"
        A1[ready_to_ship] --> B1[shipping] --> C1[shipped]
        B1 -.-> D1[order.received<br/>order.delivering<br/>order.delivered]
    end
    
    subgraph "Самовывоз"
        A2[ready_to_ship] --> C2[shipped]
        A2 -.-> D2[order.pickup_code_created<br/>order.delivered от мерчанта]
    end
    
    subgraph "Доставка мерчанта"
        A3[ready_to_ship] --> B3[shipping] --> C3[shipped]
        B3 -.-> D3[order.estimated_delivery_time<br/>order.delivering<br/>order.delivered от мерчанта]
    end
    
    style A1 fill:#e8f5e8
    style A2 fill:#e3f2fd
    style A3 fill:#fff3e0
    style B1 fill:#e8f5e8
    style B3 fill:#fff3e0
    style C1 fill:#c8e6c9
    style C2 fill:#c8e6c9
    style C3 fill:#c8e6c9
```

## Критические моменты

### ⚠️ Важные правила

1. **order.created**: Всегда отвечать успешно, даже если заказ нельзя обработать
2. **order.ready_for_delivery**: Обязательно передавать итоговый состав заказа
3. **Отмены**: Указывать причину в поле `cancellation`
4. **Таймауты**: Купер отменяет заказ при отсутствии `order.accepted` в установленное время (только для ресторанов)
5. **Аутентификация**: Поддерживать авторизацию входящих вебхуков
6. **Порядок событий**: Соблюдать правильную последовательность уведомлений

### 📋 Обязательные поля ответов

**order.created response:**
```json
{
  "status": "created",
  "number": "ID_заказа_в_системе_мерчанта",
  "expectedAssemblyTime": "2021-05-24T13:45:00+03:00" // опционально
}
```

**order.ready_for_delivery notification:**
```json
{
  "event_type": "order.ready_for_delivery",
  "payload": {
    "orderUUID": "...",
    "number": "...",
    "positions": [...], // ОБЯЗАТЕЛЬНО: итоговый состав
    "total": {...}      // ОБЯЗАТЕЛЬНО: итоговая сумма
  }
}
```

**order.delivered notification (для самовывоза/доставки мерчанта):**
```json
{
  "event_type": "order.delivered",
  "payload": {
    "orderUUID": "...",
    "number": "...",
    "positions": [...], // ОБЯЗАТЕЛЬНО: итоговый состав выданного
    "total": {...}      // ОБЯЗАТЕЛЬНО: итоговая сумма выданного
  }
}
```

**order.canceled notification:**
```json
{
  "event_type": "order.canceled",
  "payload": {
    "orderUUID": "...",
    "number": "...",
    "cancellation": {
      "slug": "reason_code",
      "description": "Описание причины отмены"
    }
  }
}
``` 
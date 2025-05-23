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
    
    ready_to_ship --> shipping : Курьер забрал заказ
    ready_to_ship --> canceled : order.canceled
    
    shipping --> shipped : order.delivered / доставлен
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

## Жизненный цикл событий

### Для типа "Сборка мерчанта, доставка Купера"

```mermaid
timeline
    title Временная шкала событий заказа
    
    section Создание
        Клиент оформляет заказ : Заказ в Купере
        order.created webhook : Уведомление мерчанту
        Response с ID мерчанта : Подтверждение от мерчанта
        
    section Подготовка  
        order.accepted (опционально) : Для ресторанов
        order.in_work notification : Начало сборки
        Поиск курьера : Купер ищет курьера
        
    section Сборка
        order.assembled notification : Заказ собран
        order.ready_for_delivery : Готов к выдаче + состав
        order.paid webhook : Оплата списана
        
    section Доставка
        courier.assigned webhook : Курьер назначен (опционально)
        courier.arrived webhook : Курьер прибыл (опционально)  
        order.received webhook : Курьер забрал заказ
        order.delivering webhook : Доставка началась
        order.delivered webhook : Заказ доставлен
```

## Обработка ошибок и отмен

```mermaid
flowchart TD
    A[Любой этап процесса] --> B{Ошибка/отмена?}
    B -->|Отмена клиентом| C[Купер отправляет order.canceled]
    B -->|Отмена Купером| D[Купер отправляет order.canceled]  
    B -->|Отмена мерчантом| E[Мерчант отправляет order.canceled]
    B -->|Технические проблемы| F[Мерчант отправляет order.canceled]
    
    C --> G[Статус: canceled]
    D --> G
    E --> G
    F --> G
    
    G --> H[Возврат средств клиенту]
    H --> I[Заказ завершен]
    
    style C fill:#ffcdd2
    style D fill:#ffcdd2  
    style E fill:#ffcdd2
    style F fill:#ffcdd2
    style G fill:#ffcdd2
```

## Особенности для разных типов интеграции

### Сборка мерчанта, самовывоз

```mermaid
flowchart LR
    A[order.ready_for_delivery] --> B[order.pickup_code_created]
    B --> C[Клиент приходит за заказом]
    C --> D[order.delivered от мерчанта]
    D --> E[Заказ завершен]
    
    style B fill:#e3f2fd
    style D fill:#e8f5e8
```

### Сборка мерчанта, доставка мерчанта

```mermaid
flowchart LR
    A[order.ready_for_delivery] --> B[order.estimated_delivery_time]
    B --> C[order.delivering от мерчанта]
    C --> D[order.delivered от мерчанта]
    D --> E[Заказ завершен]
    
    style B fill:#e8f5e8
    style C fill:#e8f5e8
    style D fill:#e8f5e8
```

## Критические моменты

### ⚠️ Важные правила

1. **order.created**: Всегда отвечать успешно, даже если заказ нельзя обработать
2. **order.ready_for_delivery**: Обязательно передавать итоговый состав заказа
3. **Отмены**: Указывать причину в поле `cancellation`
4. **Таймауты**: Купер отменяет заказ при отсутствии `order.accepted` в установленное время
5. **Аутентификация**: Поддерживать авторизацию входящих вебхуков

### 📋 Обязательные поля ответов

**order.created response:**
```json
{
  "status": "created",
  "number": "ID_заказа_в_системе_мерчанта",
  "expectedAssemblyTime": "2021-05-24T13:45:00+03:00" // опционально
}
```

**order.canceled notification:**
```json
{
  "cancellation": {
    "slug": "reason_code",
    "description": "Описание причины отмены"
  }
}
``` 
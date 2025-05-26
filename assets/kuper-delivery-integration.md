# Интеграция с доставкой Купера - Купер API

Детальное описание интеграции типа **"Сборка мерчанта, доставка Купера"** для Push-модели Купер API.

## 📋 Описание процесса

При интеграции с доставкой Купера:
- **Мерчант** собирает заказ в своем магазине
- **Купер** организует доставку через своих курьеров
- **Купер** отслеживает процесс доставки от начала до конца
- **Мерчант** получает уведомления о статусе курьера и доставки

## 📱 Детальная схема обмена сообщениями

```mermaid
sequenceDiagram
    participant Client as Клиент
    participant Kuper as Купер API
    participant PharmAPI as API Ecom
    participant OneS as 1С Розница (Аптека)
    participant Pharmacist as Фармацевт
    participant Courier as Курьер
    
    Client->>Kuper: Заказ лекарств с доставкой
    
    Kuper->>PharmAPI: webhook: order.created
    
    %% Проверка препаратов
    PharmAPI->>OneS: Проверка остатков + сроки годности
    OneS->>PharmAPI: Остатки препаратов с годностью
    
    alt Препараты доступны
        PharmAPI->>OneS: Резервирование
        OneS->>PharmAPI: Подтверждение резерва
        PharmAPI->>Kuper: response: status created, number PHARMACY_ORDER_ID
        
        Note over Pharmacist: Фармацевт начинает сборку
        Pharmacist->>OneS: Фармацевт начинает сборку
        OneS->>PharmAPI: notification: order.going_to
        PharmAPI->>Kuper: notification: order.in_work
        Note over Kuper: Начинает поиск курьера
        
        opt Курьер назначен
            Kuper->>PharmAPI: webhook: courier.assigned + данные курьера
        end
        
        %% Сборка препаратов
        loop Сборка каждого препарата
            Pharmacist->>OneS: Сборка препарата по серии
        end
        
        OneS->>PharmAPI: Документ создан + итоговый состав
        PharmAPI->>Kuper: notification: order.ready_for_delivery + итоговый состав
        
        Note over Kuper: Списывает фактическую сумму со счета клиента
        
        Kuper->>PharmAPI: webhook: order.paid
        Note over PharmAPI: Получает подтверждение оплаты

        opt Курьер прибыл в аптеку
            Kuper->>PharmAPI: webhook: courier.arrived
        end
        
        %% Передача курьеру
        Pharmacist->>Courier: Передача препаратов курьеру
        Kuper->>PharmAPI: webhook: order.received
        
        Kuper->>PharmAPI: webhook: order.delivering
        Note over Courier: Курьер доставляет заказ
        
        Kuper->>PharmAPI: webhook: order.delivered
        Note over Client: Заказ доставлен клиенту
        
    else Препараты недоступны
        PharmAPI->>Kuper: response: status created, number PHARMACY_ORDER_ID
        PharmAPI->>Kuper: notification: order.canceled + причина
        Note over Kuper: Возврат средств клиенту
    end
    
    opt Отмена заказа на любом этапе
        alt Отмена со стороны Купера/клиента
            Kuper->>PharmAPI: webhook: order.canceled + причина
            PharmAPI->>OneS: Снятие резерва/возврат товаров
        else Отмена со стороны мерчанта
            PharmAPI->>Kuper: notification: order.canceled + причина
            PharmAPI->>OneS: Снятие резерва/возврат товаров
        end
        Note over Kuper: Возврат средств клиенту
    end
```

## 🎯 Временная шкала событий

```mermaid
timeline
    title Жизненный цикл заказа с доставкой Купера
    
    section Создание и принятие
        Заказ создан : Клиент выбрал доставку
                     : order.created → response
        Принятие : order.accepted (для ресторанов)
                 : Заказ зафиксирован в системе мерчанта
        
    section Поиск курьера и сборка
        Начало сборки : order.in_work
                     : Купер начинает поиск курьера
                     : Статус заказа: collecting
        Курьер назначен : courier.assigned (опционально)
                       : Информация о курьере
        Процесс сборки : Мерчант собирает позиции
                      : Корректировка состава при необходимости
        
    section Готовность и оплата
        Готовность : order.ready_for_delivery + итоговый состав
                   : Статус заказа: ready_to_ship
        Списание оплаты : Купер списывает фактическую сумму
                       : Статус оплаты: paid
        Подтверждение оплаты : order.paid (webhook)
                            : Деньги зарезервированы за заказом
        Курьер прибыл : courier.arrived (опционально)
                     : Курьер готов забрать заказ
        
    section Забор и доставка
        Забор заказа : order.received
                    : Курьер забрал заказ у мерчанта
                    : Статус заказа: shipping
        Доставка : order.delivering
                 : Курьер едет к клиенту
        Завершение : order.delivered
                  : Заказ доставлен клиенту
                  : Статус заказа: shipped
```

## 📋 Ключевые особенности доставки Купера

### ✅ Особенности курьерской службы:

1. **courier.assigned** - Назначение курьера
   - Приходит после `order.in_work`, до `order.paid`
   - Содержит информацию о курьере (имя, телефон, возможно фото)
   - Может не прийти, если курьер назначается в последний момент

2. **courier.arrived** - Прибытие курьера
   - Приходит после `courier.assigned`, до `order.paid`
   - Означает, что курьер находится в магазине и готов забрать заказ
   - Помогает мерчанту понять, что нужно ускорить сборку

3. **order.received** - Забор заказа
   - Приходит строго после `order.paid`
   - Означает, что курьер забрал заказ и покинул магазин
   - С этого момента заказ находится у курьера

4. **order.delivering** - Процесс доставки
   - Курьер едет к клиенту
   - Клиент может отслеживать курьера в приложении

5. **order.delivered** - Завершение доставки
   - Курьер доставил заказ клиенту
   - Заказ полностью завершен

### 🚚 Логистические особенности:

- Купер самостоятельно управляет курьерами
- Мерчант не контролирует процесс доставки
- Все вебхуки о доставке приходят от Купера
- Время доставки зависит от загрузки курьерской службы

### 📦 Процесс передачи заказа курьеру:

1. Курьер приходит в магазин (может прийти `courier.arrived`)
2. Мерчант завершает сборку и отправляет `order.ready_for_delivery`
3. Купер списывает оплату и отправляет `order.paid`
4. Курьер забирает заказ (приходит `order.received`)
5. Начинается доставка (приходит `order.delivering`)

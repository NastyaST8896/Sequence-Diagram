# Агрегатор заказов и службы доставки
## **GET запрос** 
Участники процесса: клиент, сервер, база данных.

## Диаграмма последовательности для GET меотда

```mermaid

sequenceDiagram
    actor C as Клиент
    participant S as Сервер
    participant BD@{ "alias" : "База данных", "type" : "database"} 
    C->>+S: 1. GET /orders/{id} (см. табллицы 1 и 2)
    S->>S: 2. Валидация order_id
    
    
    break Ошибка валидации
    rect rgba(250, 102, 102, 0.55)
    S-->>C: 3. 400 Bad Request
    end
    end
    S->>S: 4.Аутентификация
    break Токен невалидный \ отсутствует \ истек
    rect rgba(250, 102, 102, 0.55)
    S-->>C: 5. 401 Unauthorized
    end
    end
    S->>+BD: 6. Отправка сформированного запроса
    alt пользователь не владеет заказом или заказ отсутствует
    rect rgba(250, 102, 102, 0.55)
    BD-->>S: 7. пустой результат (NULL)
    S-->>C: 8. 404 Not Found
    end
    else Ошибка подключения
    rect rgba(250, 102, 102, 0.55)
    BD-->>S:  9. Ошибка подключения
    S-->>C: 10. 500 Internal Server Error
    end
    else заказ найден
    rect rgba(117, 252, 96, 0.54)
    BD-->>-S: 11. сырые данные (Result Set)
    end

     S->>S: 12. Мапинг
    Note over S: Result Set -> JSON
    rect rgba(117, 252, 96, 0.54)
    S-->>-C: 13. 200 OK, JSON объект в теле ответа (см. таблицу 3)
    end
    end
```

```plantuml
@startuml
actor C as "Клиент"
participant S as "Сервер"
database DB as "База данных"

C->S: 1. GET /orders/{id} (см. табллицы 1 и 2)
    S->S: 2. Валидация order_id
    
    break #ff0d0069 Ошибка валидации 
    S-->C: 3. 400 Bad Request
    end
    S->S: 4.Аутентификация
    break #ff0d0069 Токен невалидный \ отсутствует \ истек
    S-->C: 5. 401 Unauthorized
    end
    S->BD: 6. Отправка сформированного запроса
    alt #ff0d0069 пользователь не владеет заказом или заказ отсутствует
    BD-->S: 7. пустой результат (NULL)
    S-->C: 8. 404 Not Found
    else Ошибка подключения
    BD-->S:  9. Ошибка подключения
    S-->C: 10. 500 Internal Server Error
    else #66e30087 заказ найден
    BD-->S: 11. сырые данные (Result Set)
     S->S: 12. Мапинг
    Note over S: Result Set -> JSON
    
    S-->C: 13. 200 OK, JSON объект в теле ответа (см. таблицу 3)
    end
@enduml
```

## 1. Таблица входных параметров (Input)
| Имя параметра | Тип данных | Обязательность | Описание |
| :--- | :--- | :--- | :--- |
| *order_id* | *String* | *Да* | *Идентификатор заказа* |

## 2. Таблица заголовков
| Имя параметра | Тип данных | Обязательность | Описание |
| :--- | :--- | :--- | :--- |
| *Authorization* | *String(Bearer Tocken)* | *Да* | *Токен доступа* |
| *Accept* | *String* | *Нет* | *Желаемый формат ответа* |
| *X-Request-ID* | *String* | *Нет* | *Идентификатор запроса* |

## 3. Таблица выходных параметров (Output)
| Имя параметра | Тип данных | Описание |
| :--- | :--- | :--- |
| *order* | *Object* | *Основная информация о заказе* |
| *order.id* | *String* | *Идентификатор заказа* |
| *order.status* | *String* | *Статус заказа* |
| *order.total* | *Number* | *Итоговая стоимость заказа* |
| *order.items* | *Array[Object]* | *Массив товаров в заказе* |
| *items[].id* | *String* | *Идентификатор товара* |
| *items[].name* | *String* | *Наименование товара* |
| *items[].quantity* | *String* | *Количество товара* |
| *items[].price* | *Number* | *Цена за единицу товара* |
| *delivery* | *Object* | *Данные о резервировании* |
| *delivery.slot_reservation_id* | *String* | *Идентификатор полученный на этапе резервирования слота доставки* |
| *delivery.tracking_number* | *String* | *Трекер-номер для отслеживания заказа* |




## **POST запрос**
Участники процесса: клиент, сервер, база данных, API службы доставки.

## Диаграмма последовательности для POST меотда
```mermaid
sequenceDiagram
    actor C as Клиент
    participant S as Сервис заказов
    participant DB@{ "alias" : "База данных", "type" : "database" }
    participant API as API службы доставки

    Note over S: 1. Получение POST запроса от Клиента
    C->>+S: POST /orders (данные заказа) (см. таблицы 1 и 2)

    Note over S: 2. Подтверждение личности пользоователя
    S->>S: Аутентификация
        break Токен невалидный \ отсутствует \ истек
            rect rgba(250, 102, 102, 0.55)
                S-->>C: 401 Unauthorized (см. таблицу 3)
            end
        end

    Note over S: 3. Валидация входных данных
    S->>S: Валидация 
        break Входные данные не валидны
            rect rgba(250, 102, 102, 0.55)
                S-->>C: 400 Bad Request (см. таблицу 4)
            end
        end

    Note over S: 4. Сохранение заказа в Базе данных и получение order_id
    S->>+DB: Создание нового заказа

    break Ошибка подключения
        rect rgba(250, 102, 102, 0.55)
            DB-->>S: Сервер временно недоступен
            S-->>C: 500 Bed Request
        end
    end

    rect rgba(117, 252, 96, 0.54)
        DB-->>-S: order_id
    end

    Note over S: 5. Резервирование слота доставки
    S->>S: count = 0, deadline = now + 5s
    loop Максимум 3 попытки, пока время < deadline
        S->>S: count++
        S->>+API: POST /reserve (order_id, ...)
        alt Клиентская ошибка (4xx)
        break 
            rect rgba(250, 102, 102, 0.55)
                API-->>S: 4xx
                S->>+DB: Обновляем статус заказа на failed
                DB-->>-S: order_id
            end
            rect rgba(117, 252, 96, 0.54)
                S -->>C: 201 Created {order, delivery} где order.status = failed
            end
            Note right of C: Выход из цикла (отказ, повтор бесполезен)
        end
        else Временная ошибка (5xx)
            API-->>S: 503 Service Unavailable (внешний сервис временно перегружен)
        
            S->>S: delay = (2^сount)*0.5с + random(0,0.3)
            S->>S: sleep(delay)

        else Успех (2xx)
            rect rgba(117, 252, 96, 0.54)
                API-->>-S: 200 OK (slot_reservation_id, tracking_number)
                S->>+DB: Обновляем статус заказа на reserved
                DB-->>-S: order_id
            end
            rect rgba(117, 252, 96, 0.54)
                S -->>C: 201 Created {order, delivery} где order.status = reserved
            end
            Note right of C: Выход из цикла (успех)
        end
    end

    Note over S:6. count = 3 или время вышло
        rect rgba(250, 102, 102, 0.55)
            S->>+DB: Обновляем статус заказа на failed
            DB-->>-S: order_id
        end
        rect rgba(117, 252, 96, 0.54)
        S -->>-C: 201 Created {order, delivery} где order.status = failed (см. таблицу 5)
    end    
```

## 1. Таблица входных параметров (Input)
| Имя параметра | Тип данных | Обязательность | Описание |
| :--- | :--- | :--- | :--- |
| *comment* | *String* | *Нет* | *Комментарий к заказу* |
| *items* | *Array[Object]* | *Да* | *Массив товаров в заказе* |
| *items[].id* | *String* | *Да* | *Идентификатор товара* |
| *items[].quantity* | *String* | *Да* | *Количество товара* |


## 2. Таблица заголовков
| Имя заголовка | Тип данных | Обязательность | Описание |
| :--- | :--- | :--- | :--- |
| *Authorization* | *String(Bearer Token)* | *Да* | *Токен доступа* |
| *Accept* | *String* | *Нет* | *Желаемый формат ответа* |
| *X-Request-ID* | *String* | *Нет* | *Идентификатор запроса* |


## 3. Таблица ошибок авторизации (HTTP 401 Unauthorized)
| Заголовок | Условие ошибки | Error | Message
| :--- | :--- | :--- | :--- |
| *Authorization* | *Отсутствует или не в формате Bearer<токен>* | *MISSING_AUTH_HEADER* | *Обязателен и должен быть в формате Bearer<токен>* |
| *Authorizations* | *Токен истек* | *EXPIRED_AUTH_HEADER* | *Срок действия токена авторизации истек* |


## 4. Таблица ошибок валидации (HTTP 400 Bad Request)
| Имя параметра | Условие ошибки | Error | Message
| :--- | :--- | :--- | :--- |
| *items* | *Отсутствует или не является массивом* | *MISSING_ITEMS* | *Массив items обязателен* |
| *items* | *Массив пустой* | *EMPTY_ITEMS* | *Массив items должен содержать хотя бы 1 товар* |
| *items[].id* | *Отсутствует, путсая строка* | *MISSING_ITEM_ID* | *item[].id  обязателен и должен быть не пустой строкой* |
| *items[].id* | *Длина > 50* | *INVALID_ITEM_ID* | *item[].id  должен быть строкой не более 50 символов* |
| *items[].quantity* | *Отсутствует* | *MISSING_ITEM_QUANTITY* | *item[].quantity обязателен* |
| *items[].quantity* | *Не число, не целое, значение < 1* | *INVALID_ITEM_QUANTITY* | *item[].quantity должен быть целым числом (не менее 1)* |

## 5. Таблица выходных параметров (Output)
| Имя параметра | Тип данных | Описание |
| :--- | :--- | :--- |
| *order* | *Object* | *Основная информация о заказе* |
| *order.id* | *String* | *Идентификатор заказа* |
| *order.status* | *String* | *Статус заказа* |
| *order.total* | *Number* | *Итоговая стоимость заказа* |
| *order.items* | *Array[Object]* | *Массив товаров в заказе* |
| *items[].id* | *String* | *Идентификатор товара* |
| *items[].name* | *String* | *Наименование товара* |
| *items[].quantity* | *String* | *Количество товара* |
| *items[].price* | *Number* | *Цена за единицу товара* |
| *delivery* | *Object* | *Данные о резервировании* |
| *delivery.slot_reservation_id* | *String* | *Идентификатор полученный на этапе резервирования слота доставки* |
| *delivery.tracking_number* | *String* | *Трекер-номер для отслеживания заказа* |

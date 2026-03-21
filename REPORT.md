# Отчёт по лабораторной работе №3
## Диагностика и оптимизация маркетплейса

**Студент:** _[ФИО]_  
**Группа:** _[Группа]_  
**Дата:** _[Дата]_

## 1. Исходные данные
### 1.1 Использованная схема
_TODO: Укажите, какую схему из предыдущей лабораторной (обычно `backend/migrations/001_init.sql`) вы использовали (файл/коммит/версия)._

### 1.2 Объём данных
_TODO: Приведите количество записей после генерации._

Пример:
- users: 10000
- orders: 100000
- order_items: 400000
- order_status_history: 199904

## 2. Найденные медленные запросы (до оптимизации)
_TODO: Укажите не менее 3 запросов, которые вы считаете медленными._

Для каждого запроса:
1. SQL текста запроса;
2. план `EXPLAIN ANALYZE` (кратко: ключевые узлы);
3. `Execution Time`;
4. почему запрос медленный.

### Запрос №1

SELECT DISTINCT user_id FROM (
  SELECT user_id, created_at
  FROM orders
  WHERE total_amount > 500
  ORDER BY created_at DESC
  LIMIT 10000
) t;

Seq Scan on orders
Sort (top-N heapsort)
Limit
HashAggregate

23.801 ms

- План оптимизирован на уровне запроса

### Запрос №2

SELECT *
FROM orders
WHERE status = 'paid'
  AND created_at >= timestamp '2025-01-01'
  AND created_at < timestamp '2025-04-01';

Seq Scan on orders

6.298 ms

- План оптимизирован на уровне запроса

### Запрос №3

SELECT SUM(oi.price * oi.quantity) AS total_revenue, o.status
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.created_at >= timestamp '2025-01-01'
  AND o.created_at < timestamp '2025-04-01'
  AND oi.price < 100
GROUP BY status
ORDER BY total_revenue DESC
LIMIT 2;

Seq Scan on order_items
Seq Scan on orders
Hash Join
HashAggregate
Sort (top-N heapsort)
Limit

35.322 ms

## 3. Добавленные индексы и обоснование типа
_TODO: Для каждого добавленного индекса опишите, почему выбран именно этот тип (`BTREE`, `BRIN`, `GIN`, partial и т.д.)._

### Индекс №1
- SQL:
CREATE INDEX IF NOT EXISTS idx_orders_amt_gt_500_created_at_desc
  ON orders (created_at DESC)
  INCLUDE (user_id)
  WHERE total_amount > 500;
- Какой запрос ускоряет: 1
- Почему выбран тип: классический BTREE для ускорения сортировки + partial для where

### Индекс №2
- SQL:
CREATE INDEX IF NOT EXISTS idx_orders_status_created_at_desc
  ON orders (status, created_at DESC);
- Какой запрос ускоряет: 2 и 3
- Почему выбран тип: BTREE никаких особых ограничеий кроме сортировок

### Индекс №3
- SQL:
CREATE INDEX IF NOT EXISTS idx_order_items_price_gt_100_orderid_include
  ON order_items (order_id)
  INCLUDE (price, quantity)
  WHERE price < 100;
- Какой запрос ускоряет: 3
- Почему выбран тип: классический BTREE для ускорения сортировки + partial для where

## 4. Замеры до/после индексов
_TODO: Заполните таблицу или список сравнений._

Пример формата:
- Query 1: до 23.801 ms, после 4.536 ms, ускорение 81%
- Query 2: до 6.298 ms, после 2.399 ms, ускорение 62%
- Query 3: до 35.322 ms, после 16.291 ms, ускорение 54%

## 5. Партиционирование `orders` по дате
### 5.1 Выбранная стратегия
RANGE по кварталам

### 5.2 Реализация
- Подготовка структуры
- Создание партиций

### 5.3 Проверка эффекта
Всё ускорилось

## 6. Итоговые замеры (после партиционирования)
- Query 1: до 4.536 ms, после 7.426 ms, замедление 64%
- Query 2: до 2.399 ms, после 2.442 ms, замедление 1.8%
- Query 3: до 16.291 ms, после 11.270 ms, ускорение 31%

## 7. Что удалось исправить
Особо тяжёлый запрос удалось ускорить, однако более быстрые несколько пострадали

## 8. Что не удалось исправить только индексами
-

Подсказки:
- full scan при высокой доле выбираемых строк;
- тяжёлые `GROUP BY`/`ORDER BY` на большом объёме;
- необходимость переписывания запроса или pre-aggregation.

## 9. Выводы
Были выполнены все задания, тяжёлый запрос удалось значительно ускорить, остальные несильно.

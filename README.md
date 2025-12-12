# Тестовое задание по ClickHouse

## Задание

Ниже тестовое задание. Предложи и объясни свое решение.

**ClickHouse** — это высокопроизводительная БД для анализа больших данных.  
В однохостовом кластере есть две таблицы по 100 млн+ строк каждая.  
Два раза в день обновляется около 1 млн товаров (на 2 порядка меньше, чем общее число строк).

### Схемы таблиц
```sql
CREATE TABLE products (
    product_id Int32,
    product_name String,
    brand_id Int32,
    seller_id Int32,
    updated Date
) ENGINE = ReplacingMergeTree
ORDER BY product_id;

CREATE TABLE remainders (
    date Date,
    product_id Int32,
    remainder Int32,
    price Int32,
    discount Int32,
    pics Int32,
    rating Int32,
    reviews Int32,
    new Bool
) ENGINE = ReplacingMergeTree
ORDER BY (date, product_id);
```

Напиши 3 причины, по которым следующий запрос выполняется медленно (5 секунд), и предложи быструю версию запроса (<0.1 секунды), без изменений архитектуры таблиц / индексов и дающую такой же результат (без дубликатов):
```sql
SELECT product_id
FROM products FINAL
JOIN remainders FINAL USING(product_id)
WHERE updated = today()
  AND date = today() - 1;
```
### Хинты
1. Первая причина — базовое понимание SQL (алгоритм фильтрации).  
2. Вторая — отличие JOIN от IN.  
3. Третья — особенности ReplacingMergeTree и FINAL.

---

## Ответ

### Порядок выполнения исходного запроса

1. Модификатор FINAL запускает Merge-on-Read для обеих таблиц, чтобы убрать дубликаты и получить актуальные версии строк.  
2. JOIN объединяет два набора по 100 млн+ строк, строя огромную хэш-таблицу в памяти.  
3. Фильтрация WHERE выполняется уже после JOIN и после применения FINAL ко всем данным.

---

## Причины медленного выполнения исходного запроса

### 1. Фильтрация происходит слишком поздно  
Условия по датам накладываются после обработки огромных объёмов данных.

### 2. JOIN вынуждает полностью читать обе таблицы  
JOIN по product_id требует загрузить все 100+ млн строк из обеих таблиц и построить хэш-структуру.

### 3. FINAL — дорогостоящая операция  
FINAL заставляет ClickHouse сканировать, декомпрессировать и сортировать все строки в таблицах ReplacingMergeTree.

---

## Быстрое решение
```sql
SELECT DISTINCT product_id
FROM remainders
WHERE date = today() - 1
  AND product_id IN (
      SELECT product_id
      FROM products
      WHERE updated = today()
  );
```
---

## Объяснение

1. Сначала подзапрос формирует хэш-сет из ~1 млн product_id.  
2. Затем таблица remainders читается только по записям за вчера — это эффективно благодаря сортировке по (date, product_id).  
3. Далее ClickHouse быстро фильтрует оставшиеся строки по IN.  
4. DISTINCT устраняет возможные дубликаты, но над существенно уменьшенным набором данных.

# SQL — COUNT & Combined Operators

## Модуль
[SQL Injection Fundamentals — SQL Operators (Advanced)](https://academy.hackthebox.com/app/module/33/section/192)

## Команда

```sql
SELECT COUNT(*) FROM titles
WHERE emp_no > 10000
OR title NOT LIKE '%engineer%';
```

**Відповідь:** `654`

## Агрегатні функції

```sql
COUNT(*)           -- кількість рядків
COUNT(column)      -- кількість NOT NULL значень
SUM(column)        -- сума
AVG(column)        -- середнє
MIN(column)        -- мінімум
MAX(column)        -- максимум
```

## Комбінація операторів

```sql
-- NOT LIKE — не містить шаблон
WHERE title NOT LIKE '%engineer%'

-- > < >= <= — порівняння чисел і дат
WHERE emp_no > 10000
WHERE hire_date < '1995-01-01'

-- OR з NOT — типова SQLi перевірка логіки
WHERE emp_no > 10000 OR title NOT LIKE '%engineer%'
```

## Важливо для SQLi

> `OR` розширює вибірку — класичний вектор:
> `' OR 1=1 --` повертає всі рядки бо умова завжди `true`

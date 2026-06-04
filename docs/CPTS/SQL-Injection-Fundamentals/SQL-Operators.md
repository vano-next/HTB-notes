# SQL — Operators & Filtering (WHERE, AND, LIKE)

## Модуль
[SQL Injection Fundamentals — SQL Operators](https://academy.hackthebox.com/app/module/33/section/191)

## Команда

```sql
SELECT last_name FROM employees
WHERE first_name LIKE 'Bar%'
AND hire_date = '1990-01-01';
```

**Відповідь:** `Mitchem`

## Оператори фільтрації

```sql
-- AND — обидві умови true
WHERE first_name = 'John' AND age > 30

-- OR — хоча б одна умова true
WHERE dept = 'IT' OR dept = 'Dev'

-- NOT — заперечення
WHERE NOT dept = 'HR'

-- LIKE — пошук по шаблону
WHERE name LIKE 'Bar%'    -- починається з Bar
WHERE name LIKE '%son'    -- закінчується на son
WHERE name LIKE '%ar%'    -- містить ar

-- BETWEEN — діапазон
WHERE hire_date BETWEEN '1990-01-01' AND '1995-12-31'

-- IN — список значень
WHERE dept_no IN ('d001', 'd002', 'd005')

-- IS NULL / IS NOT NULL
WHERE manager IS NULL
```

## Пріоритет операторів

```sql
-- AND виконується раніше OR — використовуй дужки
WHERE (dept = 'IT' OR dept = 'Dev') AND salary > 50000
```

| Символ | Значення в LIKE |
|--------|----------------|
| `%` | Будь-яка кількість символів |
| `_` | Рівно один символ |

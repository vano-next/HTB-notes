# SQL — UNION Statement

## Модуль
[SQL Injection Fundamentals — UNION Statement](https://academy.hackthebox.com/app/module/33/section/806)

## Концепція

**UNION** об'єднує результати двох SELECT запитів в одну таблицю.
Правила:
1. Кількість колонок в обох SELECT **повинна збігатись**
2. Типи даних колонок **повинні бути сумісні**
3. `UNION` видаляє дублікати, `UNION ALL` — залишає всі

## Виконані кроки

### Крок 1 — Підключення до MySQL
```bash
mysql -h TARGET_IP -P TARGET_PORT -u root -ppassword --ssl=0
```

### Крок 2 — Визначити структуру таблиць
```sql
USE employees;

DESCRIBE employees;
-- 6 колонок: emp_no, birth_date, first_name, last_name, gender, hire_date

DESCRIBE departments;
-- 2 колонки: dept_no, dept_name
```

### Крок 3 — UNION (вирівняти кількість колонок через NULL)
```sql
-- departments має 2 колонки → добавляємо NULL для решти 4
SELECT COUNT(*) FROM (
  SELECT emp_no, birth_date, first_name, last_name, gender, hire_date
  FROM employees
  UNION
  SELECT dept_no, dept_name, NULL, NULL, NULL, NULL
  FROM departments
) AS combined;
```
+----------+
| COUNT(*) |
+----------+
|      663 |
+----------+
1 row in set (0.158 sec)

**Відповідь:** `663`

---

## UNION — правила вирівнювання колонок

```sql
-- Таблиця A: 4 колонки
SELECT col1, col2, col3, col4 FROM tableA
UNION
-- Таблиця B: 2 колонки → добавляємо NULL
SELECT col1, col2, NULL, NULL FROM tableB
```

## UNION vs UNION ALL

| | UNION | UNION ALL |
|---|---|---|
| Дублікати | Видаляє | Залишає |
| Швидкість | Повільніше | Швидше |
| SQLi використання | Рідко | Часто |

## Значення для SQLi

```sql
-- Класичний UNION-based SQLi для витягування даних:
' UNION SELECT username, password, NULL, NULL FROM users-- -

-- Визначення кількості колонок через ORDER BY:
' ORDER BY 1-- -   → OK
' ORDER BY 2-- -   → OK
' ORDER BY 5-- -   → ERROR → 4 колонки
```

# SQL — SELECT Queries & Table Enumeration

## Модуль
[SQL Injection Fundamentals — SQL Statements](https://academy.hackthebox.com/app/module/33/section/190)

## Команди

```sql
-- Вибрати базу
USE employees;

-- Список таблиць
SHOW TABLES;

-- Всі записи з таблиці
SELECT * FROM departments;

-- Фільтрація по умові
SELECT * FROM departments WHERE dept_name = 'Development';

-- Вибрати конкретні колонки
SELECT dept_no, dept_name FROM departments;
```

## Результат
+---------+-------------+
| dept_no | dept_name   |
+---------+-------------+
| d005    | Development |
+---------+-------------+
1 row in set (0.013 sec)

**Відповідь:** `d005`

## Базовий синтаксис SELECT

```sql
SELECT column1, column2
FROM table_name
WHERE condition
ORDER BY column1
LIMIT 10;
```

| Клауза | Опис |
|--------|------|
| `SELECT *` | Всі колонки |
| `WHERE` | Фільтр рядків |
| `ORDER BY` | Сортування |
| `LIMIT n` | Обмежити кількість рядків |
| `LIKE '%val%'` | Пошук по шаблону |

# MySQL — Remote Connection & Basic Enumeration

## Модуль
[SQL Injection Fundamentals — MySQL Statements](https://academy.hackthebox.com/app/module/33/section/183)

## Команда підключення

```bash
mysql -h TARGET_IP -P TARGET_PORT -u root -ppassword --ssl=0
```

| Параметр | Опис |
|----------|------|
| `-h` | Target IP |
| `-P` | Порт (великий P) |
| `-u` | Username |
| `-pPASSWORD` | Пароль без пробілу після `-p` |
| `--ssl=0` | Вимкнути SSL (для старих/лаб серверів) |

## Базові команди MySQL

```sql
-- Список баз даних
SHOW DATABASES;

-- Вибрати базу
USE employees;

-- Список таблиць
SHOW TABLES;

-- Структура таблиці
DESCRIBE table_name;

-- Всі дані з таблиці
SELECT * FROM table_name;
```

## Результат
+--------------------+
| Database           |
+--------------------+
| employees          |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.013 sec)

**Відповідь:** `employees`

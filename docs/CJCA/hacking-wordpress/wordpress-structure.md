# WordPress Structure

## 🗂️ Структура файлів

WordPress за замовчуванням встановлюється у `/var/www/html/`.

```bash
tree -L 1 /var/www/html
```

```
.
├── index.php
├── license.txt
├── readme.html
├── wp-activate.php
├── wp-admin/
├── wp-blog-header.php
├── wp-comments-post.php
├── wp-config.php
├── wp-config-sample.php
├── wp-content/
├── wp-cron.php
├── wp-includes/
├── wp-links-opml.php
├── wp-load.php
├── wp-login.php
├── wp-mail.php
├── wp-settings.php
├── wp-signup.php
├── wp-trackback.php
└── xmlrpc.php
```

---

## 📌 Ключові файли

| Файл | Опис |
|---|---|
| `index.php` | Головна сторінка WordPress |
| `license.txt` | Містить версію WordPress — цікавий для розвідки |
| `readme.html` | Також розкриває версію |
| `wp-activate.php` | Активація email при реєстрації |
| `wp-config.php` | **Найцікавіший файл** — credentials БД, ключі, debug |
| `wp-config-sample.php` | Шаблон конфігу |
| `xmlrpc.php` | XML-RPC інтерфейс — потенційна точка атаки |
| `wp-cron.php` | Планувальник задач WordPress |

### Шляхи до сторінки входу

```
/wp-login.php
/wp-admin/login.php
/wp-admin/wp-login.php
/login.php
```

> ⚠️ Файл `wp-login.php` може бути перейменований адміном для приховування.

---

## ⚙️ wp-config.php

Містить усі критичні налаштування сайту:

```php
define('DB_NAME',     'database_name_here');  // Назва БД
define('DB_USER',     'username_here');        // Юзер БД
define('DB_PASSWORD', 'password_here');        // Пароль БД
define('DB_HOST',     'localhost');            // Хост БД

// Унікальні ключі та солі (для сесій та cookies)
define('AUTH_KEY',         'put your unique phrase here');
define('SECURE_AUTH_KEY',  'put your unique phrase here');
define('LOGGED_IN_KEY',    'put your unique phrase here');
define('NONCE_KEY',        'put your unique phrase here');

// Префікс таблиць у БД (за замовчуванням wp_)
$table_prefix = 'wp_';

// Режим відлагодження
define('WP_DEBUG', false);
```

```bash
# Швидке зчитування credentials (post-exploitation)
grep -E "DB_NAME|DB_USER|DB_PASSWORD|DB_HOST" /var/www/html/wp-config.php
```

---

## 📁 Ключові директорії

### `/wp-content/`

```bash
tree -L 1 /var/www/html/wp-content
# ├── index.php
# ├── plugins/    ← плагіни
# └── themes/     ← теми
```

> 🎯 `wp-content/uploads/` — завантажені файли, часто без обмежень.
> Перевіряй на наявність shell-файлів або чутливих даних.

### `/wp-includes/`

Ядро WordPress — сертифікати, шрифти, JS, віджети. Рідко цікавий при пентесті, але варто знати структуру.

```bash
tree -L 1 /var/www/html/wp-includes
```

---

## 🔎 Команди розвідки структури

```bash
# Перевірка версії через license.txt
curl -s http://TARGET/license.txt | grep -i "version"

# Перевірка версії через readme.html
curl -s http://TARGET/readme.html | grep -i "version"

# Перевірка доступності wp-config.php (зазвичай заблокований)
curl -s http://TARGET/wp-config.php

# Огляд завантажень
curl -s http://TARGET/wp-content/uploads/

# Огляд плагінів (якщо directory listing відкритий)
curl -s http://TARGET/wp-content/plugins/

# Огляд тем
curl -s http://TARGET/wp-content/themes/
```

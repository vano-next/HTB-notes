# WordPress Core Version Enumeration

## 🔍 Метод 1 — Meta Generator тег

```bash
# cURL + grep по мета-тегу
curl -s http://TARGET/ | grep 'meta name="generator"'
# Результат: <meta name="generator" content="WordPress 5.3.3" />
```

```bash
# Альтернативний grep-патерн
curl -s http://TARGET/ | grep -i 'content="WordPress'
```

> 💡 В браузері: `CTRL+U` → сторінка коду → `CTRL+F` → шукати `generator`

---

## 🔍 Метод 2 — CSS посилання

```bash
curl -s http://TARGET/ | grep -oP 'ver=[\d.]+'  | head -5
# Результат: ver=5.3.3
```

CSS файли в `wp-content/themes/` та JS файли в `wp-includes/` містять
параметр `?ver=X.X.X` — це і є версія WordPress.

```bash
# Повний grep по CSS/JS лінках
curl -s http://TARGET/ | grep -E "wp-content|wp-includes" | grep -oP 'ver=[\d.]+' | sort -u
```

---

## 🔍 Метод 3 — readme.html (старіші версії)

```bash
curl -s http://TARGET/readme.html | grep -i "version"
```

> ⚠️ У нових версіях WordPress readme.html вже не містить точну версію,
> але файл досі існує і підтверджує наявність WordPress на сервері.

---

## 🔍 Метод 4 — RSS Feed

```bash
curl -s http://TARGET/?feed=rss2 | grep '<generator>'
# Результат: <generator>https://wordpress.org/?v=5.3.3</generator>
```

---

## 🔍 Метод 5 — Колонтитул сторінки

```bash
curl -s http://TARGET/ | grep -i "wordpress" | tail -5
```

---

## 🔍 Метод 6 — /wp-login.php

```bash
curl -s http://TARGET/wp-login.php | grep -i "ver="
```

---

## 📊 Зведена таблиця методів

| Метод | Команда | Надійність |
|---|---|---|
| `meta generator` | `grep 'meta name="generator"'` | ⭐⭐⭐⭐⭐ |
| CSS/JS `?ver=` | `grep -oP 'ver=[\d.]+'` | ⭐⭐⭐⭐ |
| `readme.html` | `curl .../readme.html` | ⭐⭐⭐ |
| RSS feed | `curl .../?feed=rss2` | ⭐⭐⭐ |
| `/wp-login.php` | `grep -i "ver="` | ⭐⭐ |

---

## 🚀 Швидкий скрипт — всі методи одразу

```bash
TARGET="http://TARGET"

echo "[*] Meta Generator:"
curl -s $TARGET/ | grep 'meta name="generator"'

echo "[*] CSS/JS version:"
curl -s $TARGET/ | grep -oP 'ver=[\d.]+' | sort -u | head -3

echo "[*] readme.html:"
curl -s $TARGET/readme.html | grep -i "version" | head -3

echo "[*] RSS Feed:"
curl -s "$TARGET/?feed=rss2" | grep '<generator>'
```

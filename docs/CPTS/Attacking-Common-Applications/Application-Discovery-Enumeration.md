# Attacking Common Applications — Application Discovery & Enumeration

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Application Discovery & Enumeration](https://academy.hackthebox.com/app/module/113/section/1088)
Ціль: 10.129.81.26
Q1 Відповідь: `ew.db`
Q2 Відповідь: `Pages by Similarity`

## Концепція

При розвідці цільової інфраструктури pentester рідко обмежується одним хостом — типово в scope потрапляють десятки чи сотні веб-сервісів на різних портах, і вручну відкривати кожен в браузері неефективно. Для цього існує клас інструментів **web screenshot enumeration**: `EyeWitness` та `Aquatone` — обидва беруть на вхід результати nmap-скану (XML) і автоматично роблять скріншоти кожного знайденого веб-сервісу, групуючи їх у зручний HTML-звіт. Це критично для швидкої візуальної оцінки великої кількості цілей (which host runs what — CMS, admin panel, default page тощо) без ручного перегляду кожного порту.

## Крок 1 — Nmap-скан типових веб-портів

```bash
sudo nmap -p 80,443,8000,8080,8180,8888,10000 --open -oA web_discovery 10.129.81.26
```

Пояснення: `--open` показує лише відкриті порти, `-oA web_discovery` зберігає результат одразу в трьох форматах (`.nmap`, `.gnmap`, `.xml`) — саме `.xml`-файл потрібен обом інструментам (EyeWitness і Aquatone) як вхідні дані. На цій цілі відкритим виявився лише порт 80/tcp.

## Крок 2 (для Q1) — Запуск EyeWitness

Оскільки `eyewitness.sh` не існує в кореневій директорії (типова плутанина — скрипт для встановлення залежностей лежить у `setup/`, а сам інструмент запускається напряму через Python), запускаємо коректно:

```bash
cd ~/EyeWitness
source eyewitness-venv/bin/activate

python Python/EyeWitness.py -x ~/aquatone_scan/web_discovery.xml -d inlanefreight_eyewitness
```

Пояснення прапорців:
- `-x web_discovery.xml` — вказує EyeWitness парсити результати nmap у форматі XML, автоматично витягуючи всі відкриті веб-порти.
- `-d inlanefreight_eyewitness` — задає ім'я вихідної директорії для звіту (за потреби EyeWitness сам створить її).

## Крок 3 (для Q1) — Знаходимо .db файл

```bash
find inlanefreight_eyewitness -iname "*.db"
```

Результат: `inlanefreight_eyewitness/ew.db`

Пояснення: EyeWitness зберігає всі метадані сканування (URL, заголовки, шляхи до скріншотів, статус-коди) у локальну SQLite-базу `ew.db` всередині вихідної папки — це дозволяє повторно відкрити/фільтрувати результати без повторного сканування, а також програмно обробити дані через `sqlite3`.

**Відповідь Q1: `ew.db`**

## Крок 4 (для Q2) — Встановлення Aquatone

```bash
mkdir ~/aquatone_scan && cd ~/aquatone_scan
wget https://github.com/michenriksen/aquatone/releases/download/v1.7.0/aquatone_linux_amd64_1.7.0.zip
unzip aquatone_linux_amd64_1.7.0.zip
chmod +x aquatone
```

## Крок 5 (для Q2) — Запуск Aquatone через pipe з nmap XML

```bash
sudo nmap -p 80,443,8000,8080,8180,8888,10000 --open -oA web_discovery 10.129.81.26
cat web_discovery.xml | ./aquatone -nmap
```

Пояснення: на відміну від EyeWitness (прапорець `-x`), Aquatone приймає nmap XML через **stdin** (`cat ... | aquatone -nmap`), а не як прямий аргумент шляху до файлу — важлива відмінність у синтаксисі між двома інструментами, яку варто запам'ятати, щоб не витрачати час на пошук неіснуючого прапорця.

## Крок 6 (для Q2) — Перевірка шляху до звіту і відкриття в браузері

```bash
realpath aquatone_report.html
firefox aquatone_report.html
```

## Крок 7 (для Q2) — Читаємо заголовок титульної сторінки

Після відкриття `aquatone_report.html` в браузері на титульній сторінці звіту відображається заголовок кластеризації сторінок за візуальною схожістю.

**Відповідь Q2: `Pages by Similarity`**

## Порівняння інструментів (для нотаток)

| Параметр | EyeWitness | Aquatone |
|---|---|---|
| Вхідні дані | `-x file.xml` (прямий аргумент) | `cat file.xml \| aquatone -nmap` (stdin) |
| Формат зберігання даних | SQLite (`ew.db`) | Session file (JSON) |
| Основний вихідний файл | `report.html` | `aquatone_report.html` |
| Ключова фіча звіту | Таблиця хостів + заголовки + скріншот | Кластеризація "Pages by Similarity" |

## На що звернути увагу (типові граблі з цього прогону)

- Не існує `eyewitness.sh` в кореневій `~/EyeWitness/` — завжди перевіряйте реальну структуру репозиторію через `find . -iname "*.py" -o -iname "*.sh"`, а не покладайтесь на назву з теорії уроку буквально.
- Активуйте venv (`source eyewitness-venv/bin/activate`) **перед** запуском `EyeWitness.py` — інакше залежності (Selenium, Chrome driver) не будуть доступні.
- Aquatone і EyeWitness мають різний синтаксис подачі вхідних даних (аргумент vs stdin) — плутанина між ними одна з найчастіших причин "command not found"/"no such option" помилок на цьому уроці.

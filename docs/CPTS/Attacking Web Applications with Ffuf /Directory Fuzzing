# FFuf — Directory & File Fuzzing

**Модуль:** HTB Academy — Information Gathering / Web Fuzzing  
**Тема:** Directory Enumeration  
**Інструмент:** ffuf v2.x  

***

## Огляд

Web fuzzing — метод виявлення прихованих директорій, файлів та ендпоінтів шляхом перебору wordlist. `ffuf` (Fuzz Faster U Fool) — основний інструмент для цієї задачі в сучасному пентесті.

***

## Базова структура команди

```bash
ffuf -w <WORDLIST>:FUZZ -u http://<TARGET>/FUZZ [опції]
```

| Параметр | Опис |
|----------|------|
| `-w` | Wordlist + ім'я змінної (`:FUZZ`) |
| `-u` | URL з placeholder `FUZZ` |
| `-mc` | Match HTTP status codes (200,301,302,307,401,403) |
| `-t` | Threads (100 — стандарт для HTB) |
| `-fc` | Filter status codes (виключити) |
| `-fs` | Filter by response size (прибрати false positives) |
| `-fl` | Filter by lines count |
| `-fw` | Filter by words count |
| `-e` | Розширення файлів (`.php,.html,.txt`) |
| `-r` | Follow redirects |
| `-recursion` | Рекурсивний обхід знайдених директорій |
| `-recursion-depth` | Глибина рекурсії |
| `-o` | Вивід у файл |
| `-of` | Формат файлу (json, csv, md) |
| `-v` | Verbose (показує повний URL) |
| `-ic` | Ігнорувати коментарі у wordlist |

***

## Плейбук: Directory Enumeration

### Крок 1 — Базовий скан директорій

```bash
ffuf -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt:FUZZ \
  -u http://TARGET:PORT/FUZZ \
  -mc 200,301,302,307,401,403 \
  -t 100 \
  -ic
```

### Крок 2 — Прибрати false positives за розміром

Якщо `ffuf` повертає всі запити з однаковим розміром (наприклад `Size: 986`) — сайт повертає 200 на все. Фільтруй:

```bash
ffuf -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt:FUZZ \
  -u http://TARGET:PORT/FUZZ \
  -mc 200,301,302,307,401,403 \
  -fs 986 \
  -t 100 \
  -ic
```

> **Визначення розміру для фільтра:** запусти ffuf без фільтра, подивись на домінуючий `Size` у коментарях — це і є false positive розмір.

### Крок 3 — File fuzzing (з розширеннями)

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt:FUZZ \
  -u http://TARGET:PORT/FUZZ \
  -e .php,.html,.txt,.bak,.zip \
  -mc 200,301,302,307,401,403 \
  -t 100 \
  -ic
```

### Крок 4 — Рекурсивний обхід

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt:FUZZ \
  -u http://TARGET:PORT/FUZZ \
  -mc 200,301,302,307,401,403 \
  -t 50 \
  -recursion \
  -recursion-depth 2 \
  -ic
```

***

## Плейбук: Subdomain / VHost Fuzzing

### Subdomain enumeration

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
  -u http://FUZZ.TARGET.com \
  -mc 200,301,302,307,401,403 \
  -t 100
```

### VHost fuzzing (якщо IP відомий)

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
  -u http://TARGET_IP:PORT \
  -H "Host: FUZZ.target.com" \
  -mc 200,301,302,307,401,403 \
  -fs <default_size> \
  -t 100
```

***

## Плейбук: GET/POST Parameter Fuzzing

### GET параметри

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u "http://TARGET/page.php?FUZZ=value" \
  -mc 200 \
  -fs <default_size>
```

### POST параметри

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u http://TARGET/login.php \
  -X POST \
  -d "FUZZ=value" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mc 200 \
  -fs <default_size>
```

### Фаззинг значень параметра

```bash
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-LFISuite-pathtotest.txt:FUZZ \
  -u "http://TARGET/page.php?lang=FUZZ" \
  -mc 200 \
  -fs <default_size>
```

***

## Wordlists — що коли використовувати

| Сценарій | Wordlist |
|----------|----------|
| Швидкий скан директорій | `dirbuster/directory-list-2.3-small.txt` |
| Повний скан директорій | `seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` |
| Файли + розширення | `seclists/Discovery/Web-Content/common.txt` |
| Subdomain/VHost | `seclists/Discovery/DNS/subdomains-top1million-5000.txt` |
| Параметри | `seclists/Discovery/Web-Content/burp-parameter-names.txt` |
| API endpoints | `seclists/Discovery/Web-Content/api/objects.txt` |
| Alphanum (фаззинг символів) | `seclists/Fuzzing/alphanum-case.txt` |

```bash
# Шлях до Seclists на Kali:
/usr/share/seclists/

# Встановити якщо немає:
sudo apt install seclists
```

***

## Інтерпретація результатів

| Status | Значення |
|--------|----------|
| `200` | Сторінка існує, доступна |
| `301/302` | Redirect — директорія існує (додай `/` в кінці для перевірки) |
| `401` | Unauthorized — існує, але потрібна авторизація |
| `403` | Forbidden — існує, але доступ заборонено |
| `404` | Не існує |

### False Positives

Якщо `ffuf` повертає масово однаковий `Size` / `Words` / `Lines` — сервер відповідає 200 на все (catch-all):

```bash
# Визнач домінуючий розмір і фільтруй:
-fs 986        # filter size
-fw 423        # filter words  
-fl 56         # filter lines
```

***

## Приклад з цього ангажементу

```bash
# Ціль: 154.57.164.80:31835
ffuf -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt:FUZZ \
  -u http://154.57.164.80:31835/FUZZ \
  -mc 200,301,302,307,401,403 \
  -t 100

# Результат:
# blog    [Status: 301, Size: 322]
# forum   [Status: 301, Size: 323]
```

**Знайдені директорії:** `/blog`, `/forum`

***

## OPSEC / Рекомендації

- `-t 100` — стандарт для CTF/HTB; для реальних ангажементів знижуй до `-t 20-50`
- Додавай `-r` обережно — може змінити логіку матчингу
- Для великих сканів використовуй `-o results.json -of json` для збереження
- При рекурсії завжди обмежуй `-recursion-depth 2-3` щоб не зациклитись

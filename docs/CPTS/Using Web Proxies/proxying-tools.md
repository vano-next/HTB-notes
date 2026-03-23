# Proxying Tools

## Опис

У цій секції ми вчимося проксувати не тільки браузер, а й інші інструменти — зокрема Metasploit.  
Завдання: налаштувати Metasploit на роботу через HTTP‑проксі (Burp), відправити HTTP PUT‑запит модулем `auxiliary/scanner/http/http_put` і зчитати останній рядок цього запиту (`FILEDATA`), щоб використати його як відповідь.

---

## Середовище

- Kali / Pwnbox з:
  - Metasploit Framework (`msfconsole`).
  - Burp Suite (або ZAP, але приклад через Burp).
- Burp Proxy слухає на `127.0.0.1:8080`.  
- Цільовий хост для тесту: будь‑який веб‑сайт з HTTP на 80‑му порту (у прикладі — `example.com`).

---

## Мета

- Запустити Metasploit і вибрати модуль `auxiliary/scanner/http/http_put`.  
- Налаштувати Metasploit на використання Burp як HTTP‑проксі.  
- Відправити PUT‑запит на ціль і побачити його в Burp.  
- Визначити останній рядок HTTP‑запиту і використати його як відповідь (у нашому випадку: `msf test file`).

---

## Кроки

### 1. Запускаємо Burp Suite

```bash
burpsuite &
У майстрі запуску обираємо:

Temporary project

Use Burp defaults → Start Burp

Переконуємось, що:

Proxy → Options → Proxy Listeners містить 127.0.0.1:8080 і він увімкнений.

За бажанням очищаємо Proxy → HTTP history, щоб не заважали попередні запити.

2. Запускаємо Metasploit
bash
msfconsole
Чекаємо на промпт:

text
msf >
3. Вибір модуля http_put
bash
msf > use auxiliary/scanner/http/http_put
Перевіряємо:

bash
msf auxiliary(scanner/http/http_put) >
Дивимось доступні опції:

bash
show options
Ключові параметри:

RHOSTS — цільовий хост.

RPORT — порт (за замовчуванням 80).

FILENAME — ім’я файлу, який модуль спробує записати (msf_http_put_test.txt).

FILEDATA — вміст файлу / тіла PUT‑запиту (msf test file за замовчуванням).

PROXIES — налаштування проксі.

4. Налаштування проксі в Metasploit
Вказуємо Burp як HTTP‑проксі:

bash
set PROXIES HTTP:127.0.0.1:8080
Перевіряємо:

bash
show options
У полі Proxies тепер:

text
HTTP:127.0.0.1:8080
5. Вказуємо ціль
Для прикладу використовуємо example.com на 80‑му порту:

bash
set RHOSTS example.com
set RPORT 80
(За потреби можна вказати інший сайт, головне — HTTP на 80‑му.)

6. Запуск модуля
bash
run
Metasploit спробує зробити HTTP PUT‑запит на:

text
http://example.com/msf_http_put_test.txt
через проксі Burp.

Навіть якщо спроба запису не вдасться (404/403), сам HTTP PUT‑запит все одно піде через Burp — це те, що нам потрібно.

7. Аналіз запиту в Burp
У Burp:

Вкладка Proxy → HTTP history.

Шукаємо рядок з:

Method: PUT

Host: example.com (або інший, який ти вказав у RHOSTS).

Клікаємо по цьому рядку.

У правій частині:

Зверху — Request (режим Raw).

Знизу — Response.

Приклад запиту:

text
PUT /msf_http_put_test.txt HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.2903.86
Content-Type: text/plain
Content-Length: 13
Connection: keep-alive

msf test file
За умовами завдання, правильною відповіддю є останній рядок цього HTTP‑запиту:

text
msf test file
Саме цей текст потрібно ввести в полі відповіді в секції Proxying Tools HTB Academy.

Remediation / Нотатки
Проксі‑ланцюги (PROXIES у Metasploit) дозволяють прозоро проганяти інструменти через Burp/ZAP не змінюючи сам код інструмента.

Так можна логувати/міняти трафік будь‑яких клієнтів: Metasploit, nmap NSE‑скрипти (через proxychains), власні скрипти на Python/Go тощо.

З точки зору безпеки, адміністратори можуть застосовувати аналогічний підхід для моніторингу й фільтрації вихідного трафіку інструментів і сканерів у корпоративній мережі.

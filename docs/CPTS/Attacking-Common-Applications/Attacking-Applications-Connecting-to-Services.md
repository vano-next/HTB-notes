Attacking Common Applications — Attacking Applications Connecting to Services
=============================================================================

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Attacking Applications Connecting to Services](https://academy.hackthebox.com/app/module/113/section/2154)
Ціль: `10.129.205.20` (ACADEMY-ACA-ROLLOUT)
SSH: `htb-student / HTB_@cademy_stdnt!`
Фінальна відповідь: `SA:N0tS3cr3t!`

Ідея атаки
----------

На хості `10.129.205.20` знайдено ELF‑бінарник `octopus_checker`, який:
- через ODBC драйвер підключається до локального MS SQL (`localhost, 1401`);
- використовує SQL connection string, зашитий у коді (з кредами до БД);
- при виклику `SQLDriverConnect` тримає цей рядок у регістрі `RDX`;
- ми можемо через GDB/PEDA поставити breakpoint на `SQLDriverConnect@plt` і прочитати connection string;
- так витягуємо креденшали до локальної БД: `SA:N0tS3cr3t!`.

Крок 1 — SSH доступ до цілі
---------------------------

```bash
ssh htb-student@10.129.205.20
# пароль: HTB_@cademy_stdnt!
```

Після логіну:

```bash
pwd
# /home/htb-student
ls
# octopus_checker  peda
```

Фіксуємо:

- `octopus_checker` — цільовий ELF‑бінарник;
- `peda` — вже встановлений GDB‑плагін для зручної експлойт‑дебагінг роботи.

Крок 2 — Перший запуск octopus_checker
--------------------------------------

```bash
./octopus_checker
```

Вивід:

```text
Program had started..
Attempting Connection 
Connecting ... 

The driver reported the following diagnostics whilst running SQLDriverConnect

01000:1:5701:[Microsoft][ODBC Driver 17 for SQL Server][SQL Server]Changed database context to 'master'.
01000:2:5703:[Microsoft][ODBC Driver 17 for SQL Server][SQL Server]Changed language setting to us_english.
connected
```

Висновок:

- програма успішно конектиться до MS SQL Server через ODBC Driver 17;
- десь усередині коду є connection string з кредами до `localhost:1401`.

Крок 3 — Запускаємо GDB з PEDA
------------------------------

```bash
gdb ./octopus_checker
```

Отримаємо стандартний GDB‑промпт:

```text
Reading symbols from ./octopus_checker...
(No debugging symbols found in ./octopus_checker)
gdb-peda$
```

Опціонально можемо:

```bash
gdb-peda$ set disassembly-flavor intel
```

але для задачі з кредами це не обовʼязково.

Крок 4 — Ставимо breakpoint на SQLDriverConnect@plt
---------------------------------------------------

Адреса `SQLDriverConnect@plt` визначається/береться з уроку (у лабі — `0x5555555551b0`).

Ставимо брейкпоінт:

```bash
gdb-peda$ b *0x5555555551b0
Breakpoint 1 at 0x5555555551b0
```

Запускаємо програму:

```bash
gdb-peda$ run
```

Програма йде до моменту конекту й зупиняється на брейкпоінті.  

Крок 5 — Читаємо connection string з RDX
----------------------------------------

На брейкпоінті дивимось регістри:

```bash
gdb-peda$ i r
```

Серед іншого бачимо:

```text
RDX: 0x7fffffffdf30 ("DRIVER={ODBC Driver 17 for SQL Server};SERVER=localhost, 1401;UID=SA;PWD=N0tS3cr3t!;")
```

Рядок чітко містить:

- `UID=SA`
- `PWD=N0tS3cr3t!`

Це й є креденшали до локального SQL Server інстанса.

Крок 6 — Фіксуємо креденшали для відповіді / нотаток
----------------------------------------------------

Connection string:

```text
DRIVER={ODBC Driver 17 for SQL Server};SERVER=localhost, 1401;UID=SA;PWD=N0tS3cr3t!;
```

Креденшали:

- username: `SA`
- password: `N0tS3cr3t!`

Формат для Q1:

- `SA:N0tS3cr3t!`

Відповідь
---------

**Q1:** What credentials were found for the local database instance while debugging the octopus_checker binary? (Format username:password)  
**A:** `SA:N0tS3cr3t!`

Що тут було вразливим
---------------------

- Бінарник містить hard‑coded connection string із відкритим паролем до локальної БД.
- Відсутність захисту (обфускація, шифрування секретів, секрет‑менеджер) дозволяє через статичний/динамічний аналіз легко витягнути креденшали.
- Кред `SA` — це високопривілейований користувач SQL Server; його витік дає можливість повного контролю над БД і потенційної привілей‑ескалації в середовищі.

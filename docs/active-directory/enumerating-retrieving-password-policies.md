# Enumerating & Retrieving Password Policies

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1490](https://academy.hackthebox.com/app/module/143/section/1490) |

---

## 🔑 Ключові команди

### З Linux — з credentials

```bash
# CrackMapExec
crackmapexec smb 172.16.5.5 -u avazquez -p Password123 --pass-pol

# rpcclient
rpcclient -U "" -N 172.16.5.5
rpcclient $> querydominfo
rpcclient $> getdompwinfo
```

### З Linux — SMB NULL Session (без credentials)

```bash
# enum4linux
enum4linux -P 172.16.5.5

# enum4linux-ng (з виводом у файл)
enum4linux-ng -P 172.16.5.5 -oA ilfreight
cat ilfreight.json

# rpcclient NULL session
rpcclient -U "" -N 172.16.5.5
```

### З Linux — LDAP Anonymous Bind

```bash
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength

# Newer ldapsearch — використовуй -H замість -h
ldapsearch -H ldap://172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

### З Windows — net.exe (вбудований)

```cmd
net accounts
```

### З Windows — PowerView

```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

### З Windows — null session

```cmd
net use \\DC01\ipc$ "" /u:""
```

---

## 📊 Порти інструментів

| Інструмент | Порт |
|------------|------|
| nmblookup / nbtstat | 137/UDP |
| net | 139/TCP, 135/TCP |
| rpcclient | 135/TCP |
| smbclient / CME | 445/TCP |

---

## 📋 Default Password Policy (новий домен)

| Policy | Default Value |
|--------|---------------|
| Enforce password history | 24 days |
| Maximum password age | 42 days |
| Minimum password age | 1 day |
| **Minimum password length** | **7** |
| Password complexity | Enabled |
| Reversible encryption | Disabled |
| Lockout duration | Not set |
| **Lockout threshold** | **0** |
| Reset lockout counter | Not set |

---

## ⚠️ Правила безпечного Password Spraying

- Не більше **2-3 спроб** на акаунт за цикл
- Між циклами чекати **мінімум 31 хвилину** (lockout duration = 30 хв)
- При threshold = 3 — не більше **1 спроби** за цикл
- **Ніколи** не локаути акаунти у клієнта!

---

## ❓ Questions

### Q1 — Default мінімальна довжина пароля

> **What is the default Minimum password length when a new domain is created?**

**Answer:** `7`

---

### Q2 — minPwdLength у INLANEFREIGHT.LOCAL

> **What is the minPwdLength set to in the INLANEFREIGHT.LOCAL domain?**

**Steps:**
```bash
ssh htb-student@<TARGET_IP>   # pass: HTB_@cademy_stdnt!

# Через ldapsearch або crackmapexec:
crackmapexec smb 172.16.5.5 -u avazquez -p Password123 --pass-pol
# або
rpcclient -U "" -N 172.16.5.5
rpcclient $> getdompwinfo
# min_password_length: 8
```

**Answer:** `8`

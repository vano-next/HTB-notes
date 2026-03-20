```markdown
# Living Off the Land

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1360](https://academy.hackthebox.com/app/module/143/section/1360) |

***

## 🔑 Ключові команди

### Базова інформація про хост
```powershell
hostname
[System.Environment]::OSVersion.Version
wmic qfe get Caption,Description,HotFixID,InstalledOn
ipconfig /all
set
```

### Інформація про безпеку (антивірус/Windows Defender)
```powershell
Get-MpComputerStatus
Get-MpComputerStatus | Select AMProductVersion
```

### Члени локальної групи Administrators
```powershell
net localgroup administrators
Get-LocalGroupMember -Group "Administrators"
```

### Локальні користувачі
```powershell
Get-LocalUser
Get-LocalUser | Where-Object {$_.Enabled -eq $false} | Select Name, Description
```

### AD — відключені акаунти з описом (пошук прихованих даних)
```powershell
Get-ADUser -Filter {Enabled -eq $false} -Properties Description | `
  Where-Object {$_.Description -ne $null} | Select Name, Description
```

### AD — відключені акаунти з адмін правами
```powershell
Get-ADUser -Filter {Enabled -eq $false -and AdminCount -eq 1} `
  -Properties Description | Select Name, Description
```

### LDAP фільтри через PowerShell
```powershell
# Всі юзери де Password Can't Change NOT встановлено
(&(objectClass=user)(!userAccountControl:1.2.840.113556.1.4.803:=64))
```

***

## 🧪 Практика

### Середовище

| Параметр | Значення |
|----------|----------|
| Attack Host | `10.129.x.x` (ACADEMY-EA-MS01) |
| RDP | `htb-student` / `Academy_student_AD!` |
| DC | `172.16.5.5` (ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL) |

### Q1 — Enumerate the host's security configuration. Provide its AMProductVersion.

```powershell
Get-MpComputerStatus | Select AMProductVersion
```

```
AMProductVersion
----------------
4.18.2109.6
```

✅ **Відповідь: `4.18.2109.6`**

***

### Q2 — What domain user is explicitly listed as a member of the local Administrators group?

```powershell
net localgroup administrators
```

```
Members
-------
Administrator
INLANEFREIGHT\adunn
INLANEFREIGHT\Domain Admins
INLANEFREIGHT\Domain Users
```

✅ **Відповідь: `adunn`**

***

### Q3 — Find the flag hidden in the description field of a disabled account with administrative privileges.

```powershell
Get-ADUser -Filter {Enabled -eq $false -and AdminCount -eq 1} `
  -Properties Description | Select Name, Description
```

```
Name        Description
----        -----------
Betty Ross  HTB{LD@P_I$_W1ld}
```

✅ **Відповідь: `HTB{LD@P_I$_W1ld}`**
```

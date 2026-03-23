```markdown
# ACL Enumeration

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1485](https://academy.hackthebox.com/app/module/143/section/1485) |

***

## 🔑 Ключові команди

### Крок 1 — Підключення до цільової машини
```bash
xfreerdp /u:htb-student /p:'Academy_student_AD!' /v:<TARGET_IP>
```

### Крок 2 — Імпорт PowerView
```powershell
Import-Module C:\Tools\PowerView.ps1
```

### Крок 3 — Отримати SID цільового юзера
```powershell
# Конвертуємо ім'я юзера в SID для подальших запитів
$sid = Convert-NameToSid <username>

# Приклад
$sid = Convert-NameToSid forend
```

### Крок 4 — Знайти всі ACL права юзера (без human-readable)
```powershell
# Повертає GUID значення — важко читати
Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

### Крок 5 — Знайти всі ACL права юзера (з human-readable)
```powershell
# -ResolveGUIDs конвертує GUID в читабельний формат (напр. User-Force-Change-Password)
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

### Крок 6 — Знайти права над конкретним об'єктом
```powershell
# Перевірити права конкретного юзера над іншим юзером
Get-DomainObjectACL -ResolveGUIDs -Identity <target_user> | ? {$_.SecurityIdentifier -eq $sid}

# Перевірити права над групою
Get-DomainObjectACL -ResolveGUIDs -Identity "<group_name>" | ? {$_.SecurityIdentifier -eq $sid}
```

### Альтернатива без PowerView — через Get-Acl та Get-ADUser
```powershell
# Крок 1 — Зберегти список всіх юзерів домену
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > C:\Users\htb-student\Desktop\ad_users.txt

# Крок 2 — Перевірити ACL для кожного юзера
foreach($line in [System.IO.File]::ReadLines("C:\Users\htb-student\Desktop\ad_users.txt")) {
  get-acl "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | 
  Where-Object {$_.IdentityReference -match 'INLANEFREIGHT\\<username>'}
}
```

### Конвертувати GUID в читабельну назву права
```powershell
$guid = "00299570-246d-11d0-a768-00aa006e0529"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" `
  -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | `
  Select Name,DisplayName,DistinguishedName,rightsGuid | `
  Where-Object {$_.rightsGuid -eq $guid} | fl
```

***

## 🧪 Практика

### Середовище

| Параметр | Значення |
|----------|----------|
| Attack Host | `10.129.x.x` (ACADEMY-EA-MS01) |
| RDP | `htb-student` / `Academy_student_AD!` |
| DC | `172.16.5.5` (ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL) |

### Q1 — What is the rights GUID for User-Force-Change-Password?

```powershell
# Конвертуємо GUID в читабельну назву права
$guid = "00299570-246d-11d0-a768-00aa006e0529"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" `
  -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | `
  Select Name,DisplayName,rightsGuid | Where-Object {$_.rightsGuid -eq $guid} | fl
```

```
Name      : User-Force-Change-Password
rightsGuid: 00299570-246d-11d0-a768-00aa006e0529
```

✅ **Відповідь: `00299570-246d-11d0-a768-00aa006e0529`**

***

### Q2 — What flag can we use with PowerView to show us the ObjectAceType in human-readable format?

```powershell
# Без флагу — повертає нечитабельний GUID
Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}

# З флагом — повертає читабельний формат (напр. User-Force-Change-Password)
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

✅ **Відповідь: `ResolveGUIDs`**

***

### Q3 — What privileges does the user damundsen have over the Help Desk Level 1 group?

```powershell
# Крок 1 — Отримати SID damundsen
$sid2 = Convert-NameToSid damundsen

# Крок 2 — Знайти права над групою Help Desk Level 1
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid2}
```

```
ActiveDirectoryRights : ListChildren, ReadProperty, GenericWrite
ObjectDN : CN=Help Desk Level 1,...
```

✅ **Відповідь: `GenericWrite`**

***

### Q4 — Enumerate the ActiveDirectoryRights that the user forend has over the user dpayne.

```powershell
# Крок 1 — Підключитись по RDP
# xfreerdp /u:htb-student /p:'Academy_student_AD!' /v:10.129.x.x

# Крок 2 — Імпортувати PowerView та отримати SID forend
Import-Module C:\Tools\PowerView.ps1
$sid = Convert-NameToSid forend

# Крок 3 — Перевірити права над dpayne
Get-DomainObjectACL -ResolveGUIDs -Identity dpayne | ? {$_.SecurityIdentifier -eq $sid}
```

```
ActiveDirectoryRights : GenericAll
ObjectDN : CN=Dagmar Payne,...
```

✅ **Відповідь: `GenericAll`**

***

### Q5 — What is the ObjectAceType of the first right that forend has over GPO Management group?

```powershell
# Знайти права forend над групою GPO Management
Get-DomainObjectACL -ResolveGUIDs -Identity "GPO Management" | ? {$_.SecurityIdentifier -eq $sid}
```

```
ObjectAceType : Self-Membership   ← перший результат
ActiveDirectoryRights : Self
```

✅ **Відповідь: `Self-Membership`**
```

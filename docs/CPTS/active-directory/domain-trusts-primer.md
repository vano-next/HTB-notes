# Domain Trusts Primer

## ℹ️ Informations

- **Website:** HackTheBox
- **Module:** Active Directory Enumeration & Attacks
- **Link:** https://academy.hackthebox.com/app/module/143/section/1488

---

## Теорія

Domain trust — це зв'язок між системами аутентифікації двох доменів, який дозволяє користувачам одного домену отримувати доступ до ресурсів іншого.

### Типи трастів

| Тип | Опис |
|-----|------|
| **Parent-child** | Два або більше домени в одному лісі. Child домен має двосторонній transitive trust з батьківським. Наприклад: `corp.inlanefreight.local` ↔ `inlanefreight.local` |
| **Cross-link** | Trust між child доменами для прискорення аутентифікації |
| **External** | Non-transitive trust між доменами в різних лісах, де немає forest trust. Використовує SID filtering |
| **Tree-root** | Двосторонній transitive trust між forest root та новим tree root доменом |
| **Forest** | Transitive trust між двома forest root доменами |
| **ESAE** | Bastion forest для управління Active Directory |

### Transitive vs Non-transitive

- **Transitive**: якщо A довіряє B, а B довіряє C — тоді A автоматично довіряє C
- **Non-transitive**: тільки прямий trust між двома доменами, не поширюється далі

### Напрямки трастів

- **One-way**: користувачі trusted домену можуть отримати доступ до ресурсів trusting домену, але не навпаки
- **Bidirectional**: користувачі обох доменів можуть отримати доступ до ресурсів один одного

### Scenario Setup

- RDP: `10.129.26.48` (ACADEMY-EA-MS01) | `htb-student` / `Academy_student_AD!`

---

## Enumeration Commands

### Get-ADTrust (вбудований модуль AD)

```powershell
PS C:\htb> Import-Module activedirectory
PS C:\htb> Get-ADTrust -Filter *

Direction               : BiDirectional
DistinguishedName       : CN=LOGISTICS.INLANEFREIGHT.LOCAL,...
ForestTransitive        : False
IntraForest             : True
Name                    : LOGISTICS.INLANEFREIGHT.LOCAL
Target                  : LOGISTICS.INLANEFREIGHT.LOCAL

Direction               : BiDirectional
DistinguishedName       : CN=FREIGHTLOGISTICS.LOCAL,...
ForestTransitive        : True
IntraForest             : False
Name                    : FREIGHTLOGISTICS.LOCAL
Target                  : FREIGHTLOGISTICS.LOCAL
PowerView — Get-DomainTrust
powershell
PS C:\htb> Get-DomainTrust

SourceName      : INLANEFREIGHT.LOCAL
TargetName      : LOGISTICS.INLANEFREIGHT.LOCAL
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : WITHIN_FOREST
TrustDirection  : Bidirectional

SourceName      : INLANEFREIGHT.LOCAL
TargetName      : FREIGHTLOGISTICS.LOCAL
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : FOREST_TRANSITIVE
TrustDirection  : Bidirectional
PowerView — Get-DomainTrustMapping (повна карта)
powershell
PS C:\htb> Get-DomainTrustMapping
# Показує трасти з усіх доменів, включаючи LOGISTICS та FREIGHTLOGISTICS
Перерахування користувачів child домену
powershell
PS C:\htb> Get-DomainUser -Domain LOGISTICS.INLANEFREIGHT.LOCAL | select SamAccountName

samaccountname
--------------
htb-student_adm
Administrator
Guest
lab_adm
krbtgt
netdom — класичний інструмент Windows
text
C:\htb> netdom query /domain:inlanefreight.local trust
Direction Trusted\Trusting domain        Trust type
<->       LOGISTICS.INLANEFREIGHT.LOCAL  Direct
<->       FREIGHTLOGISTICS.LOCAL         Direct

C:\htb> netdom query /domain:inlanefreight.local dc
ACADEMY-EA-DC01

C:\htb> netdom query /domain:inlanefreight.local workstation
ACADEMY-EA-MS01
ACADEMY-EA-MX01
SQL01
BloodHound — візуалізація трастів
text
Pre-built query: "Map Domain Trusts"
→ Показує INLANEFREIGHT.LOCAL ↔ LOGISTICS.INLANEFREIGHT.LOCAL (IntraForest, Bidirectional)
→ Показує INLANEFREIGHT.LOCAL ↔ FREIGHTLOGISTICS.LOCAL (Forest, Bidirectional)
❓ Question 1
What is the child domain of INLANEFREIGHT.LOCAL? (format: FQDN, i.e., DEV.ACME.LOCAL)

📋 Walkthrough
Підключення до MS01:

bash
xfreerdp /v:10.129.26.48 /u:htb-student /p:'Academy_student_AD!' /dynamic-resolution
Запустити Get-ADTrust або Get-DomainTrust:

powershell
PS C:\htb> Import-Module activedirectory
PS C:\htb> Get-ADTrust -Filter *

# Результат показує:
# IntraForest : True → це child domain
# Name        : LOGISTICS.INLANEFREIGHT.LOCAL
З виводу бачимо, що LOGISTICS.INLANEFREIGHT.LOCAL має IntraForest = True — це і є child domain в тому ж лісі.

✅ Answer: LOGISTICS.INLANEFREIGHT.LOCAL
❓ Question 2
What domain does the INLANEFREIGHT.LOCAL domain have a forest transitive trust with?

📋 Walkthrough
powershell
PS C:\htb> Get-ADTrust -Filter *

# Другий запис:
# ForestTransitive : True
# IntraForest      : False
# Name             : FREIGHTLOGISTICS.LOCAL
ForestTransitive = True вказує на forest trust (між різними лісами). Це домен FREIGHTLOGISTICS.LOCAL.

✅ Answer: FREIGHTLOGISTICS.LOCAL
❓ Question 3
What direction is this trust?

📋 Walkthrough
powershell
PS C:\htb> Get-DomainTrust

SourceName      : INLANEFREIGHT.LOCAL
TargetName      : FREIGHTLOGISTICS.LOCAL
TrustAttributes : FOREST_TRANSITIVE
TrustDirection  : Bidirectional
Trust між INLANEFREIGHT.LOCAL та FREIGHTLOGISTICS.LOCAL має напрямок Bidirectional — обидва домени довіряють один одному.

✅ Answer: Bidirectional

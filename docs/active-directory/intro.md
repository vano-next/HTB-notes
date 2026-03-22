# Active Directory Enumeration & Attacks

## Вступ до Active Directory

**Active Directory (AD)** — служба каталогів для Windows-підприємств, що була офіційно реалізована в 2000 році з випуском Windows Server 2000.

### Основні характеристики AD:

- Базується на протоколах **x.500** та **LDAP**
- Розподілена, ієрархічна структура
- Централізоване управління ресурсами організації
- Надає функції:
  - **Аутентифікації** (Authentication)
  - **Обліку** (Accounting)  
  - **Авторизації** (Authorization)

### Чому важливо знати AD?

- 📊 **43%** ринкової частки серед рішень Identity and Access Management
- 🔴 **2000+** CVE вразливостей Microsoft за останні два роки
- 🎯 Неправильні конфігурації сервісів і дозволів — основна причина вразливостей

---

## Реальні Attack Chains

### Сценарій 1: Waiting On An Admin

```
SYSTEM на domain-joined host
  ↓
Kerberoasting (отримання TGS tickets)
  ↓
Hashcat + d3ad0ne rule overnight
  ↓
Cleartext password користувача (write access на shares)
  ↓
SCF файли на shares + Responder
  ↓
NetNTLMv2 hash Domain Admin
  ↓
Domain compromise
```

### Сценарій 2: Spraying The Night Away

```
SMB NULL session (enum4linux)
  ↓
Список користувачів + password policy
  ↓
Password spraying (Spring@18)
  ↓
BloodHound enumeration
  ↓
Local admin на хості з Domain Admin сесією
  ↓
Rubeus - витягнення Kerberos TGT
  ↓
Pass-the-ticket → Domain Admin
  ↓
Nested group membership → Trusting domain takeover
```

### Сценарій 3: Fighting In The Dark

```
Kerbrute username enumeration (linkedin2username)
  ↓
516 валідних користувачів
  ↓
Password spraying (Welcome2021) - 1 hit
  ↓
BloodHound.py - RDP доступ до host
  ↓
DomainPasswordSpray (Fall2021) - кілька hits
  ↓
Help Desk user → GenericAll на Enterprise Key Admins group
  ↓
Enterprise Key Admins → GenericAll на DC
  ↓
Shadow Credentials attack → NT hash DC machine account
  ↓
DCSync → всі NTLM hashes домену
```

---

## Практичне середовище

### Windows Attack Host (MS01)

**Підключення через RDP:**

```bash
xfreerdp /v:<TARGET_IP> /u:htb-student /p:Academy_student_AD!
```

- Інструменти: `C:\Tools`
- PowerShell модулі автоматично завантажуються

### Parrot Linux Attack Host (ATTACK01)

**Підключення через SSH:**

```bash
ssh htb-student@<TARGET_IP>
```

**Підключення через RDP (XRDP):**

```bash
xfreerdp /v:<TARGET_IP> /u:htb-student /p:HTB_@cademy_stdnt!
```

- Інструменти: встановлені або у `/opt`
- BloodHound GUI доступний через RDP

---

## Корисні посилання

- [HTB Academy - Active Directory Module](https://academy.hackthebox.com/module/143)
- [Introduction to Active Directory](https://academy.hackthebox.com/course/preview/introduction-to-active-directory)

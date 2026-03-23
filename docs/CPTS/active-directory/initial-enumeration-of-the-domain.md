# Initial Enumeration of the Domain

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1265](https://academy.hackthebox.com/app/module/143/section/1265) |

---

## 🔌 Підключення до лаби

```bash
# SSH до Linux attack host
ssh htb-student@<TARGET_IP>
# Password: HTB_@cademy_stdnt!

# Або через RDP (Windows attack host)
xfreerdp /v:<TARGET_IP> /u:htb-student /p:'Academy_student_AD!' /cert:ignore
```

---

## 🎯 Key Data Points (що шукаємо)

| Data Point | Опис |
|------------|------|
| `AD Users` | Валідні акаунти для password spraying |
| `AD Joined Computers` | DC, file servers, SQL, web, Exchange |
| `Key Services` | Kerberos (88), NetBIOS (139), LDAP (389), DNS (53) |
| `Vulnerable Hosts` | Legacy OS, unpatched services — швидкий foothold |

---

## 🔑 Ключові команди

### 1. Пасивне прослуховування мережі

```bash
# Wireshark (з GUI)
sudo -E wireshark

# TCPDump (без GUI) — зберегти у файл для подальшого аналізу
sudo tcpdump -i ens224
sudo tcpdump -i ens224 -w capture.pcap

# Responder у режимі аналізу (ПАСИВНИЙ — не отруює мережу)
sudo responder -I ens224 -A
```

> 💡 З ARP/MDNS пакетів можна дізнатися IP та імена хостів ще до будь-якого активного сканування

---

### 2. Активне виявлення хостів

```bash
# fping — швидкий ICMP sweep підмережі
fping -asgq 172.16.5.0/23
# -a → показати живі хости
# -s → статистика в кінці
# -g → генерувати список з CIDR
# -q → не спамити результатами кожного хосту

# Зберегти результат у файл для Nmap
fping -asgq 172.16.5.0/23 2>&1 | tee hosts.txt
```

---

### 3. Nmap сканування

```bash
# Сканування всіх знайдених хостів
sudo nmap -v -A -iL hosts.txt -oA /home/htb-student/Documents/host-enum

# Детальне сканування одного хосту
nmap -A 172.16.5.5

# Фокус на AD-портах
sudo nmap -p 53,88,135,139,389,445,464,636,3268,3269,3389 -sV 172.16.5.5

# Зберігати завжди у всіх форматах!
# -oA = -oN (normal) + -oX (xml) + -oG (grepable)
```

> ⚠️ `nmap -A` запускає scripted scans — може тригерити IDS або дестабілізувати legacy системи

---

### 4. Kerbrute — Enumeration AD акаунтів

```bash
# Клонувати та скомпілювати
sudo git clone https://github.com/ropnop/kerbrute.git
cd kerbrute && sudo make all

# Перемістити у PATH
sudo mv dist/kerbrute_linux_amd64 /usr/local/bin/kerbrute

# Завантажити username wordlist
# https://github.com/insidetrust/statistically-likely-usernames

# Запустити enumeration
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 jsmith.txt -o valid_ad_users
```

> 💡 Kerbrute використовує Kerberos pre-auth failures — **не тригерить event logs** (stealth)  
> ⚠️ Провалені спроби **рахуються як failed logins** — може заблокувати акаунти!

---

### 5. Ідентифікація Domain Controller

З виводу Nmap на 172.16.5.5:

```
PORT   SERVICE
53    DNS
88    Kerberos
389   LDAP  → Domain: INLANEFREIGHT.LOCAL
445   SMB
3268  Global Catalog LDAP
```

```
commonName = ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

---

### 6. Виявлення вразливих legacy хостів

```bash
nmap -A 172.16.5.100
```

Ознаки legacy OS у виводі Nmap:
- `Windows Server 2008 R2` → вразливий до EternalBlue, MS17-010
- `IIS 7.5` → застаріла версія
- `SQL Server 2008 R2 RTM` → без патчів

> ⚠️ Перед експлуатацією legacy систем — **отримати письмовий дозвіл клієнта!**

---

### 7. Способи отримати SYSTEM на domain-joined хості

| Метод | Опис |
|-------|------|
| MS08-067 / EternalBlue / BlueKeep | Remote exploits для Windows |
| Juicy Potato | SeImpersonate через service account |
| Windows 10 Task Scheduler 0-day | Local PrivEsc |
| PsExec як local admin | `psexec /s cmd.exe` → SYSTEM |

З SYSTEM на domain-joined хості можна:
- Запускати BloodHound, PowerView
- Виконувати Kerberoasting / AS-REP Roasting
- Збирати Net-NTLMv2 через Inveigh
- Token impersonation для захоплення domain user

---

## ❓ Questions

### Q1 — commonName хосту 172.16.5.5

> **From your scans, what is the "commonName" of host 172.16.5.5?**

**Steps:**

```bash
# SSH до attack host
ssh htb-student@<TARGET_IP>  # pass: HTB_@cademy_stdnt!

# Сканування хосту
sudo nmap -A 172.16.5.5

# У виводі шукаємо ssl-cert або rdp-ntlm-info:
# | Subject Alternative Name: DNS:ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
# | DNS_Computer_Name: ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

**Answer:** `ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL`

---

### Q2 — Хост з Microsoft SQL Server 2019 15.00.2000.00

> **What host is running "Microsoft SQL Server 2019 15.00.2000.00"? (IP address, not Resolved name)**

**Steps:**

```bash
# Спочатку знайти всі живі хости
fping -asgq 172.16.5.0/23 2>&1 | grep -v "ICMP" > hosts.txt

# Сканувати з фокусом на MSSQL (порт 1433)
sudo nmap -p 1433 -sV -iL hosts.txt | grep -B5 "2019"

# Або повне сканування
sudo nmap -v -A -iL hosts.txt -oA host-enum
grep -A5 "15.00.2000" host-enum.nmap
```

**Answer:** `172.16.5.130`

---

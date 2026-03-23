```markdown
# Credentialed Enumeration - from Linux

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1269](https://academy.hackthebox.com/app/module/143/section/1269) |

***

## 🔑 Ключові команди

### CrackMapExec — enumerate користувачів домену
```bash
crackmapexec smb <DC_IP> --users -u <user> -p '<password>'
```

### CrackMapExec — enumerate груп домену (з membercount)
```bash
crackmapexec smb <DC_IP> --groups -u <user> -p '<password>'
```

### CrackMapExec — enumerate shares
```bash
crackmapexec smb <DC_IP> --shares -u <user> -p '<password>'
```

### rpcclient — знайти юзера за RID
```bash
# Конвертуй RID: decimal → hex (напр. 1170 = 0x492)
rpcclient -U "user%password" <DC_IP> -c "queryuser 0x<HEX_RID>"
```

### rpcclient — enumerate всіх юзерів
```bash
rpcclient -U "user%password" <DC_IP> -c "enumdomusers"
```

### rpcclient — enumerate груп та їх членів
```bash
rpcclient -U "user%password" <DC_IP> -c "enumdomgroups"
rpcclient -U "user%password" <DC_IP> -c "querygroupmem 0x<GROUP_RID>"
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

### Q1 — What AD User has a RID equal to Decimal 1170?

```bash
# 1170 decimal = 0x492 hex
rpcclient -U "forend%Klmcargo2" 172.16.5.5 -c "queryuser 0x492"
```

```
User Name   :   mmorgan
user_rid    :   0x492
```

✅ **Відповідь: `mmorgan`**

***

### Q2 — What is the membercount of the "Interns" group?

```bash
crackmapexec smb 172.16.5.5 -u forend -p 'Klmcargo2' --groups 2>&1 | grep -i "interns"
```

```
Interns   membercount: 10
```

✅ **Відповідь: `10`**
```

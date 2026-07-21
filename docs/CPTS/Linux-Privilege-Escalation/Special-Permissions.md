# Special Permissions

Ціль: `10.129.114.137`  

Питання:
- Q1 — Find a file with the setuid bit set that was not shown in the section command output (full path to the binary).
- Q2 — Find a file with the setgid bit set that was not shown in the section command output (full path to the binary).

Відповіді:
- Q1 — `/bin/sed`
- Q2 — `/usr/bin/facter`

***

## Крок 1 — Підключення до цілі

```bash
ssh htb-student@10.129.114.137
```

- Підключення до машини `10.129.114.137` під користувачем `htb-student`.
- Пароль: `Academy_LLPE!`.

Після входу ми в стандартному bash на Ubuntu 18.04 (`NIX02`).

***

## Крок 2 — Пошук файлів з setuid (SUID) для Q1

### Теорія

- **Setuid (SUID)** – спеціальний біт прав на виконуваних файлах.
- Якщо setuid увімкнено на файлі, який належить `root`, програму, запущену користувачем, процес виконує з правами `root`.
- У правовому полі `ls -l` це виглядає як `s` замість `x` у блоці прав **власника**:

  ```text
  -rwsr-xr-x 1 root root ... /usr/bin/passwd
  ```

  Тут `rws` означає, що на позиції виконання (`x`) стоїть `s` – setuid.

### Команда для SUID-пошуку

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

Розбір по поличках:

- `find /` – скануємо **всю файлову систему** з кореня `/`.
- `-user root` – беремо файли, власник яких `root`.
- `-perm -4000` – шукаємо файли з **setuid-бітом** (4000).
- `-exec ls -ldb {} \;` – для кожного знайденого файла:
  - запускаємо `ls` з ключами:
    - `-l` – детальний список з правами;
    - `-d` – показати сам файл/директорію, а не його вміст;
    - `-b` – екранувати спецсимволи в імені.
  - `{}` – поточний файл, який знайшов `find`.
- `2>/dev/null` – помилки доступу (Permission denied) викидаємо в `/dev/null`.

### Результат (фрагмент)

```text
-rwsr-xr-x 1 root root 109000 Jan 30  2018 /bin/sed
...
```

Файл `/bin/sed`:

- власник `root`;
- права `-rwsr-xr-x` – `s` у блоці власника = **setuid**;
- повний шлях: `/bin/sed`.

Це і є потрібний setuid-файл для **Q1**:

> **Q1:** `/bin/sed`

***

## Крок 3 — Пошук файлів з setgid (SGID) для Q2

### Теорія

- **Setgid (SGID)** – спеціальний біт для виконуваних файлів і директорій.
- На виконуваних файлах:
  - процес запускається з **групою власника файлу**, а не групою користувача.
- У виводі `ls -l` це `s` у блоці прав **групи**:

  ```text
  -rwxr-sr-x 1 root root ... /usr/bin/somebinary
  ```

  `r-s` у групі означає setgid.

### Команда для SUID+SGID-пошуку

```bash
find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null
```

Розбір:

- `-perm -6000` – шукаємо файли, де встановлені біти:
  - `4000` – setuid;
  - `2000` – setgid;
  - разом `6000` – **будь-яка комбінація** цих двох (часто обидва).

### Результат (фрагмент)

```text
-rwsr-sr-x 1 root root 130264 May 29  2023 /usr/lib/snapd/snap-confine
-rwsr-sr-x 1 root root 227520 Mar 19  2018 /usr/bin/facter
```

Візьмемо `/usr/bin/facter`:

- права: `-rwsr-sr-x`;
- власник: `root`, група: `root`;
- `s` і у власника, і у групи → стоять **обидва спецбіти**: setuid та setgid;
- повний шлях: `/usr/bin/facter`.

Це правильна відповідь для **Q2**:

> **Q2:** `/usr/bin/facter`

***

## Конспект команд (щоб кидати в нотатки)

### Пошук SUID-файлів

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

- `/` – скан всієї системи.
- `-user root` – тільки файли `root`.
- `-perm -4000` – setuid.
- `-exec ls -ldb {} \;` – показати файл з правами.

### Пошук SUID+SGID-файлів

```bash
find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null
```

- `-perm -6000` – будь-яке з комбінації setuid/setgid.

### Відповіді:

- **Q1:** `/bin/sed`
- **Q2:** `/usr/bin/facter`

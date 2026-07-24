## Shared Object Hijacking

### Лінк

- [HTB Academy – Linux Privilege Escalation, Shared Object Hijacking](https://academy.hackthebox.com/app/module/51/section/476) [intrusionz3r0.gitbook](https://intrusionz3r0.gitbook.io/intrusionz3r0/linux-penetration-testing/privilege-escalation/shared-object-hijacking)

### Тема уроку

Shared Object Hijacking – зловживання динамічними бібліотеками (`.so`) для привілейованого виконання коду.

### Ціль

- Навчитися:
  - знаходити залежності бінарників через `ldd`; [notes.dollarboysushil](https://notes.dollarboysushil.com/linux-privilege-escalation/shared-object-manipulation)
  - дивитися інформацію про ELF та секції бібліотек через `readelf`; [notes.dollarboysushil](https://notes.dollarboysushil.com/linux-privilege-escalation/shared-object-manipulation)
  - розуміти порядок пошуку shared‑об’єктів системою і як це можна використати для прівеску (наприклад, підмінити бібліотеку в записуваній директорії або змінити `LD_LIBRARY_PATH`). [boiteaklou](https://www.boiteaklou.fr/Abusing-Shared-Libraries.html)
- Для конкретного питання модуля:
  - визначити версію **glibc** на машині (`ldd --version` / `libc.so.6`) і здати її у форматі `X.Y` (тут – `2.27`). [linuxconfig](https://linuxconfig.org/how-to-check-libc-library-version-on-debian-linux)

***

## Питання секції (Q1)

> Submit the version of glibc (i.e. 2.30) in use to move on to the next section.

Ти виконав:

```bash
ldd --version
# ldd (Ubuntu GLIBC 2.27-3ubuntu1.6) 2.27
...
```

і перевірив шлях до libc:

```bash
ldd "$(which bash)" | grep libc
# libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f72f677d000)
```

З першого рядка `ldd --version` видно:

- реалізація – `Ubuntu GLIBC 2.27-3ubuntu1.6`;
- «чистий» номер версії – **`2.27`**. [dev](https://dev.to/bitecode/how-to-get-glibc-version-c-lang-26he)

Отже, відповідь для Academy:

> `2.27`

***

## Мікрошпаргалка: як перевіряти версію glibc

1. **Через `ldd --version` (найпростіший спосіб):** [linuxconfig](https://linuxconfig.org/how-to-check-libc-library-version-on-debian-linux)

   ```bash
   ldd --version
   ```

   Перший рядок містить версію glibc, наприклад:

   ```text
   ldd (Ubuntu GLIBC 2.27-3ubuntu1.6) 2.27
   ```

2. **Через саму `libc.so.6`:**

   Спочатку знайти шлях:

   ```bash
   ldd "$(which bash)" | grep libc
   # libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 ...
   ```

   Потім запустити:

   ```bash
   /lib/x86_64-linux-gnu/libc.so.6
   # GNU C Library (Ubuntu GLIBC 2.27-3ubuntu1.6) stable release version 2.27.
   ```

3. **Програмно (C‑код з `gnu_get_libc_version`) – потрібно рідше, більше для розуміння:** [stackoverflow](https://stackoverflow.com/questions/9705660/check-glibc-version-for-a-particular-gcc-compiler)

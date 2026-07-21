## Sudo Rights Abuse

Ціль: `10.129.114.137`  

Питання:
- What command can the htb-student user run as root?

Відповідь:
- `/usr/bin/openssl`

***

## Крок 1 — Підключення на NIX02

```bash
ssh htb-student@10.129.114.137
```

- Логін на хост `10.129.114.137` під користувачем `htb-student`.
- Пароль: `Academy_LLPE!`.

Після входу ти на машині `NIX02` (Ubuntu 18.04.6) у звичайному shell.

***

## Крок 2 — Перевірити sudo-права

### Команда:

```bash
sudo -l
```

**Що робить:**

- `sudo -l` показує, які команди поточний користувач може виконувати через `sudo`, і з чиїми правами.
- Виводить:
  - глобальні `Defaults` (типу `env_reset`, `secure_path`),
  - список дозволених команд.

Фрагмент виводу:

```text
Matching Defaults entries for htb-student on NIX02:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    env_keep+=LD_PRELOAD

User htb-student may run the following commands on NIX02:
    (root) NOPASSWD: /usr/bin/openssl
```

***

## Крок 3 — Інтерпретація sudo-рядка

```text
(root) NOPASSWD: /usr/bin/openssl
```

Розбір:

- `(root)` – команда буде виконана **з правами користувача root**.
- `NOPASSWD:` – виконання **без запиту пароля**; тобто `htb-student` може викликати її як root, просто через `sudo`, не вводячи пароль.
- `/usr/bin/openssl` – **конкретний бінарник**, який дозволено запускати.

Отже, правильна відповідь на питання секції:

> **What command can the htb-student user run as root?**  
> `/usr/bin/openssl`

***

## Конспект для нотаток (Sudo Rights Abuse)

1. Підключитися:

   ```bash
   ssh htb-student@10.129.114.137
   # Password: Academy_LLPE!
   ```

2. Подивитися sudo-права:

   ```bash
   sudo -l
   ```

3. Знайти рядок:

   ```text
   (root) NOPASSWD: /usr/bin/openssl
   ```

4. Висновок:

   - `htb-student` може як root запускати:

     ```bash
     sudo /usr/bin/openssl
     ```

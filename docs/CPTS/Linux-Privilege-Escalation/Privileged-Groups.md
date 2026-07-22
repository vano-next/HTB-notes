## Privileged Groups

Ціль: `10.129.2.210`  

Питання:
- Use the privileged group rights of the `secaudit` user to locate a flag.

Відповідь:
- `ch3ck_th0se_gr0uP_m3mb3erSh1Ps!`

***

## Крок 1 — Підключення як secaudit

```bash
ssh secaudit@10.129.2.210
# Password: Academy_LLPE!
```

Після логіну ти на `NIX02` як користувач `secaudit`.

***

## Крок 2 — Подивитися групи користувача

```bash
id
```

Результат:

```text
uid=1010(secaudit) gid=1010(secaudit) groups=1010(secaudit),4(adm)
```

- `secaudit` входить до групи **`adm`**.
- Група `adm` має доступ до **логів системи** (зазвичай `/var/log`, `/var/log/apache2` тощо).

***

## Крок 3 — Перевірити права на /var/log

```bash
ls -l /var/log
```

Ти бачиш багато файлів/каталогів, де група – `adm`, а права `r` для групи:

```text
drwxr-x---  2 root   adm  ... apache2
-rw-r-----  1 syslog adm  ... auth.log
...
-rw-r-----  1 syslog adm  ... syslog
...
drwxr-x---  2 root   adm  ... unattended-upgrades
...
```

Це означає:

- Файли/каталоги, де група `adm`, читабельні (`r`) для учасників `adm`.
- `secaudit` може зайти в каталоги типу `/var/log/apache2` і читати їхні логи — це і є «privileged group rights».

***

## Крок 4 — Зайти в лог Apache

```bash
cd /var/log/apache2
ls
```

Результат:

```text
access.log  error.log  other_vhosts_access.log
```

Тут лежить HTTP-лог `access.log`, до якого має доступ група `adm`, отже й користувач `secaudit`.

***

## Крок 5 — Знайти флаг через grep

### Команда:

```bash
grep -ri flag access.log
```

Розбір:

- `grep` – пошук рядків за шаблоном.
- `-r` – рекурсивний пошук (для каталогу; тут ми вказуємо конкретний файл, але ключ не заважає).
- `-i` – ігнорувати регістр (шукати `flag`, `FLAG`, `Flag` тощо).
- `flag` – слово-шаблон.
- `access.log` – Apache access log, де може бути захований текст флага.

Результат:

```text
10.10.14.3 - - [01/Sep/2020:05:34:22 +0200] "GET /flag%20=%20ch3ck_th0se_gr0uP_m3mb3erSh1Ps! HTTP/1.1" 301 409 "-" "Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0"
10.10.14.3 - - [01/Sep/2020:05:34:22 +0200] "GET /flag%20=%20ch3ck_th0se_gr0uP_m3mb3erSh1Ps HTTP/1.1" 404 27847 "-" "Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0"
```

У першому рядку:

- `GET /flag%20=%20ch3ck_th0se_gr0uP_m3mb3erSh1Ps! HTTP/1.1`
- `%20` – URL-encoded пробіл, отже всередині запиту:

  ```text
  /flag = ch3ck_th0se_gr0uP_m3mb3erSh1Ps!
  ```

Правильний флаг:  

> `ch3ck_th0se_gr0uP_m3mb3erSh1Ps!`

***

## Конспект команд для Privileged Groups

1. Логін:

   ```bash
   ssh secaudit@10.129.2.210
   # Password: Academy_LLPE!
   ```

2. Перевірити групи:

   ```bash
   id
   # ... groups=1010(secaudit),4(adm)
   ```

   - Бачимо, що користувач у групі `adm`.

3. Перейти у /var/log і каталог Apache:

   ```bash
   cd /var/log/apache2
   ls
   # access.log error.log other_vhosts_access.log
   ```

4. Шукати флаг у логах:

   ```bash
   grep -ri flag access.log
   ```

   - Вихід дає HTTP-запит, де значення `flag = ch3ck_th0se_gr0uP_m3mb3erSh1Ps!`.

5. Відповідь для модуля:

   ```text
   ch3ck_th0se_gr0uP_m3mb3erSh1Ps!
   ```

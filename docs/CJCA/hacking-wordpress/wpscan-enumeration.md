# WPScan Enumeration

## Базовий запуск

```bash
wpscan --url http://TARGET --api-token YOUR_TOKEN
Енумерація плагінів, тем і юзерів
bash
# Усе за замовчуванням
wpscan --url http://TARGET --enumerate --api-token YOUR_TOKEN

# Усі плагіни
wpscan --url http://TARGET --enumerate ap --api-token YOUR_TOKEN

# Вразливі плагіни
wpscan --url http://TARGET --enumerate vp --api-token YOUR_TOKEN

# Теми
wpscan --url http://TARGET --enumerate at --api-token YOUR_TOKEN

# Користувачі
wpscan --url http://TARGET --enumerate u --api-token YOUR_TOKEN
Приклад пошуку версії photo-gallery
bash
wpscan --url $TARGET --enumerate ap --api-token YOUR_TOKEN | grep -i -A5 "photo-gallery"
У виводі шукаємо рядок типу:

text
[+] photo-gallery
 | Location: http://TARGET/wp-content/plugins/photo-gallery/
 | Latest version: 1.5.34
Відповідь у HTB: 1.5.34.

# Encoding/Decoding

## Опис

У цій секції ми працюємо не з живою веб‑ціллю, а з файлом, який містить кількаразово закодований рядок.  
Завдання: послідовно декодувати його різними методами (base64, URL‑encoding тощо), поки не отримаємо прапор формату `HTB{...}`.

---

## Середовище

- Локальний Kali / Pwnbox з доступом до браузера та інструментів.  
- Файл `encoded_flag.zip`, завантажений з секції Encoding/Decoding в HTB Academy.  
- Інструменти:
  - Консольні утиліти: `wget`, `unzip`, `cat`, `base64`.  
  - (Опційно) Burp Decoder або ZAP Encoder/Decoder/Hash для наочного декодування.

---

## Мета

- Завантажити архів з Academy.  
- Витягнути текстовий файл з закодованим рядком.  
- Поетапно декодувати цей рядок до отримання валідного прапора.  
- Внести прапор як відповідь у секції Encoding/Decoding.

---

## Кроки

### 1. Підготовка робочої директорії

```bash
mkdir -p ~/htb/using-web-proxies-encoding
cd ~/htb/using-web-proxies-encoding
2. Завантаження архіву з Academy
У секції Encoding/Decoding копіюємо посилання з кнопки Download і використовуємо його в wget:

bash
wget "https://academy.hackthebox.com/storage/modules/110/encoded_flag.zip"
ls
Очікуваний результат:

text
encoded_flag.zip
3. Розпакування архіву
bash
unzip encoded_flag.zip
ls
Має з’явитися текстовий файл з encoded‑рядком:

text
encoded_flag.txt  encoded_flag.zip
4. Перегляд вмісту
bash
cat encoded_flag.txt
Приклад вмісту:

text
VTJ4U1VrNUZjRlZXVkVKTFZrWkdOVk5zVW10aFZYQlZWRmh3UzFaR2NITlRiRkphWld0d1ZWUllaRXRXUm10M1UyeFNUbVZGY0ZWWGJYaExWa1V3ZVZOc1VsZGlWWEJWVjIxNFMxWkZNVFJUYkZKaFlrVndWVmR0YUV0V1JUQjNVMnhTYTJGM1BUMD0=
Це виглядає як base64.

Декодування (приклад через CLI)
У модулі показують декодування через Burp/ZAP, але логіка така сама — багаторазове декодування, поки не отримаємо прапор.

1. Перше декодування base64
bash
cat encoded_flag.txt | base64 -d
Результат знову виглядає як base64 → повторюємо.

2. Повторні декодування
Повторюємо команду кілька разів (або копіюємо кожен проміжний рядок і знову пропускаємо через base64 -d), доки замість безглуздого набору символів не з’явиться щось схоже на URL‑кодування або звичайний рядок.

З практики цього завдання:

Кілька (4) послідовних декодувань base64.

Потім один декод URL‑кодування (якщо є %‑послідовності).

Приклад URL‑декоду через Python:

bash
echo 'URL_КОДОВАНИЙ_РЯДОК' | python3 -c "import sys,urllib.parse;print(urllib.parse.unquote(sys.stdin.read().strip()))"
Результат
У фіналі отримуємо прапор формату:

text
HTB{3nc0d1n6_n1nj4}
Саме це значення вводимо в полі відповіді секції Encoding/Decoding в HTB Academy.

Remediation / Нотатки
Encoding/decoding сам по собі не є вразливістю, але часто використовується для приховування даних у параметрах запитів, cookies або файлах.

У пентестах завжди перевіряй, чи параметри не є просто base64/URL/hex‑обгорткою навколо зрозумілих даних (JSON, SQL, конфігурація, секрети).

Ніколи не покладайся на “кодування” як на засіб безпеки — це лише перетворення формату, а не захист.

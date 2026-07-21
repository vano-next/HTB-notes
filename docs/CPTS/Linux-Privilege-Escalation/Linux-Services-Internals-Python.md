Linux Services & Internals Enumeration
====================================================

Урок: Linux Services & Internals Enumeration
Ціль: `10.129.113.212`
Питання: What is the latest Python version that is installed on the target?
Відповідь: `3.11`

Кроки:
------

1. SSH до цілі:

   ```bash
   ssh htb-student@10.129.113.212
   # Password: HTB_@cademy_stdnt!
   ```

2. Перевірити доступні версії Python у `/usr/bin`:

   ```bash
   ls /usr/bin/python*
   ```

   Вивід:

   ```text
   /usr/bin/python3  /usr/bin/python3.11  /usr/bin/python3.8
   ```

3. Визначити найновішу версію:

   - доступні `3.8` та `3.11`;
   - `3.11` > `3.8` → остання версія: `3.11`.

4. (Опціонально) підтвердити командою:

   ```bash
   python3.11 --version
   ```

   Очікувано:

   ```text
   Python 3.11.x
   ```

Відповідь:
---------

- Latest Python version installed: `3.11`

Нотатки:
--------

- `ls /usr/bin/python*` — швидка енумерація встановлених Python-бінарників.
- Знання встановленого Python важливе для подальших привілей-ескалаційних технік, що використовують `python -c`, скрипти, або специфічні експлойти, які вимагають певної версії інтерпретатора.

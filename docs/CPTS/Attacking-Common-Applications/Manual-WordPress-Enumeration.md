# Attacking Common Applications — Manual WordPress Enumeration

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Application Discovery & Enumeration](https://academy.hackthebox.com/app/module/113/section/1100)
Ціль: 10.129.81.26
Q1 Відповідь: `0ptions_ind3xeS_ftw!`
Q2 Відповідь: `WP Sitemap Page`
Q3 Відповідь: `1.6.4`

## Концепція

Ця секція демонструє класичний ланцюжок для WordPress-розвідки: **virtual host discovery → CMS fingerprinting → directory listing abuse → plugin enumeration через readme.txt**. Ключова ідея — WordPress-плагіни майже завжди залишають публічно доступний файл `readme.txt` у своїй директорії (стандарт WP.org для опису плагіна), який містить назву, версію (`Stable tag`) і changelog — це офіційний, легітимний спосіб точної версійної атрибуції без потреби брутфорсити чи гадати.

## Крок 1 — Virtual Host Discovery (fuzzing по Host-заголовку)

```bash
ffuf -u http://10.129.81.26/ -H "Host: FUZZ.inlanefreight.local" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 184
```

Пояснення: сервер може хостити кілька віртуальних хостів на одній IP (`app`, `drupal`, `dev`, `blog`), розрізняючи їх лише через заголовок `Host` — типова конфігурація Apache/Nginx name-based virtual hosting. `-fs 184` фільтрує розмір відповіді за замовчуванням (сторінка "not found" / дефолтна), залишаючи лише реальні відповіді. Знайдено `blog` — імовірний WordPress-сайт.

## Крок 2 — Реєстрація хосту локально і CMS fingerprinting

```bash
echo '10.129.81.26 blog.inlanefreight.local' | sudo tee -a /etc/hosts

curl -s http://blog.inlanefreight.local/ | grep -i "generator\|wp-content"
```

Результат підтвердив: `<meta name="generator" content="WordPress 5.8" />` та шляхи `/wp-content/plugins/contact-form-7/`, `/wp-content/plugins/mail-masta/` — WordPress з двома активними плагінами, видимими прямо в HTML head без потреби в WPScan.

## Крок 3 (Q1) — Directory Listing Abuse → flag.txt

```bash
curl -s http://blog.inlanefreight.local/wp-content/uploads/2021/08/flag.txt
```

Пояснення: назва плагіна `mail-masta` та шлях `/wp-content/uploads/2021/08/` натякнули на можливий directory listing (типова помилка конфігурації Apache — `Options +Indexes` не вимкнено для `/uploads/`). Прямий запит до передбачуваного шляху вивантажень одразу віддав файл. Сам вміст флага — `0ptions_ind3xeS_ftw!` — прямо натякає на root cause (увімкнена директива `Options Indexes`).

## Крок 4 — Перевірка readme.txt відомого плагіна (baseline-техніка)

```bash
curl -s http://blog.inlanefreight.local/wp-content/plugins/mail-masta/readme.txt
```

Це підтверджує сам метод version fingerprinting через `readme.txt` — знайдено `Mail Masta`, `Stable tag: 1.0` (плагін з відомими публічними CVE на LFI/SQLi, цінна знахідка для подальшої exploitation-фази, хоч і не є прямою відповіддю тут).

## Крок 5 (Q2 + Q3) — Пошук другого, неявного плагіна вручну

```bash
curl -s http://blog.inlanefreight.local/ | grep -i "sitemap"
curl -s http://blog.inlanefreight.local/wp-content/plugins/wp-sitemap-page/readme.txt
```

Пояснення логіки: `contact-form-7` і `mail-masta` вже видно прямо в HTML head (лінки на CSS/JS-файли плагінів), але **не всі плагіни підключають свої assets на кожній сторінці** — деякі активуються лише через шорткод у контенті (`[wp_sitemap_page]`), тому в head-тегах їх немає. Треба або переглянути повний вміст сторінок (пошук слова "sitemap" в тілі), або напряму спробувати типові назви директорій плагінів через `/wp-content/plugins/{name}/readme.txt`, ґрунтуючись на функціональних підказках із самого сайту (наявність посилання "Sitemap" в меню/футері).

Результат `readme.txt`:
```
=== WP Sitemap Page ===
...
Stable tag: 1.6.4
```

**Відповідь Q2: `WP Sitemap Page`**
**Відповідь Q3: `1.6.4`**

## Чому це працює (root cause для звіту)

- **Опис:** Веб-сервер має увімкнений `Options +Indexes` для директорії завантажень, а встановлені WordPress-плагіни зберігають публічно доступні `readme.txt` файли з точною версією.
- **Технічні деталі:** Apache без явного `Options -Indexes` в конфігурації vhost/`.htaccess` автоматично генерує листинг файлів для будь-якої директорії без `index.php`/`index.html`; стандарт WordPress.org вимагає від розробників плагінів включати `readme.txt` з полем `Stable tag`, яке ніхто не видаляє при деплої в продакшн.
- **Ризик/Імпакт:** Information Disclosure (CWE-200) — розкриття внутрішньої структури файлової системи та точних версій третьосторонніх компонентів дозволяє зловмиснику миттєво зіставити встановлені плагіни з публічними CVE (наприклад, Mail Masta 1.0 має відомі LFI/SQLi вразливості) без потреби в активному скануванні, що знижує шанси виявлення IDS/WAF.
- **Рекомендації:** Додати `Options -Indexes` на рівні vhost для всіх директорій завантажень; видаляти або блокувати доступ до `readme.txt` в `/wp-content/plugins/*/` через правило веб-сервера (`<Files readme.txt> Require all denied </Files>`); регулярно оновлювати плагіни та проводити аудит встановлених розширень.

## Швидкий playbook на майбутнє (WordPress plugin enum без WPScan)

```bash
# 1. Знайти всі плагіни, підключені в HTML head
curl -s http://TARGET/ | grep -oP '/wp-content/plugins/[^/]+' | sort -u

# 2. Для кожного — перевірити readme.txt на версію
for p in $(curl -s http://TARGET/ | grep -oP '/wp-content/plugins/[^/]+' | sort -u); do
  echo "== $p =="
  curl -s "http://TARGET$p/readme.txt" | grep -i "Stable tag"
done

# 3. Брутфорс типових/невидимих плагінів (якщо не в head)
ffuf -u "http://TARGET/wp-content/plugins/FUZZ/readme.txt" \
  -w /usr/share/seclists/Discovery/Web-Content/wp-plugins.fuzz.txt \
  -mc 200
```

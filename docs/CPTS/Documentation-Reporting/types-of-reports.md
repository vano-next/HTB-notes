## Шпаргалка – Types of Reports (HTB Academy, модуль 162, секція 1538)

### Лінк

- Урок: [https://academy.hackthebox.com/app/module/162/section/1538](https://academy.hackthebox.com/app/module/162/section/1538) – *Types of Reports / Types of Assessments & Perspectives*. [scribd](https://www.scribd.com/document/914603613/Documentation-Reporting)

***

## Тема уроку

Types of Reports: огляд типів оцінок безпеки (vulnerability assessment проти penetration test), різних перспектив тестування (black box, grey box, white box) і того, як вони впливають на формат та зміст звітів. [vaadata](https://www.vaadata.com/en/blog/black-box-penetration-testing-objective-methodology-and-use-cases/)

***

## Ціль

- Розрізняти **vulnerability assessment** vs **penetration test**:
  - VA – переважно автоматизований, без активної експлуатації;  
  - PT – ручна експлуатація, спроби отримати доступ/дані. [pentest-standard.readthedocs](https://pentest-standard.readthedocs.io/en/latest/reporting.html)
- Розуміти **black / grey / white box perspectives**:
  - Black box – **без попередньої інформації**, лише назва, домен, публічні адреси;  
  - Grey box – обмежена інформація (користувацькі креденшли, часткова дока);  
  - White box – повний доступ (схеми, код, адмін‑доступ). [hackmosphere](https://www.hackmosphere.fr/en/penetration-testing-black-box-white-box-and-gray-box/)
- Застосувати це до практичних сценаріїв Inlanefreight (Elizabeth, Nicolas) та відповісти на Q1–Q2.

***

## Список запитань

1. **Q1:** Inlanefreight замовила в Elizabeth assessment, який “mostly automated” і “no exploitation attempted”. Що це за тип ассесменту?  
2. **Q2:** Nicolas виконує external & internal penetration test; клієнт надав лише назву компанії та мережевий конекшен onsite, **без додаткових деталей**. З якої перспективи він проводить тест?

***

## Відповіді

- **Q1:** `Vulnerability Assessment`. [scribd](https://www.scribd.com/document/914603613/Documentation-Reporting)
- **Q2:** `Black Box`. [redscan](https://www.redscan.com/news/types-of-pen-testing-white-box-black-box-and-everything-in-between/)

***

## Конспект по суті уроку

### 1. Типи технічних оцінок

У модулі розрізняють, зокрема: [aqua-cloud](https://aqua-cloud.io/penetration-testing-report-templates/)

- **Vulnerability Assessment (VA):**
  - Основний акцент – **сканування вразливостей**;  
  - Переважно автоматизовані інструменти (Nessus, OpenVAS, Qualys);  
  - Зазвичай **немає реальної експлуатації** або вона строго обмежена;  
  - Звіт фокусується на переліку вразливостей, їхніх CVSS/ризику та рекомендаціях.  

- **Penetration Test (PT):**
  - Ручна **експлуатація** знайдених вразливостей для демонстрації впливу;  
  - Ітеративний процес (recon → scanning → exploitation → post‑exploitation → reporting);  
  - Звіт містить attack paths, прівеск, доступ до даних, іноді сценарії lateral movement.  

Сценарій з Elizabeth:

> “mostly automated where no exploitation is attempted”

– це чіткий опис **Vulnerability Assessment**, а не penetration test. [scribd](https://www.scribd.com/document/914603613/Documentation-Reporting)

***

### 2. Перспективи тестування (Black/Grey/White Box)

**Black Box:** [vaadata](https://www.vaadata.com/en/blog/black-box-penetration-testing-objective-methodology-and-use-cases/)

- Пентестер **не має попередніх знань** про інфраструктуру;  
- Отримує лише базову інформацію (часто – назву компанії, домен, IP діапазони);  
- Моделює реального зовнішнього атакувальника без инсайдерських даних;  
- Для internal black‑box – може бути просто підключення до внутрішньої мережі без схем, користувачів тощо.  

**Grey Box:**

- Часткові дані: стандартні креденшли, обмежена документація, окремі схеми;  
- Моделює атакувальника, який уже має певний foothold чи insider‑інфо. [hackmosphere](https://www.hackmosphere.fr/en/penetration-testing-black-box-white-box-and-gray-box/)

**White Box:**

- Повна прозорість: мережеві діаграми, дока, вихідний код, адмін‑доступ;  
- Мета – комплексний, глибокий аналіз конкретних компонентів. [pentest-standard.readthedocs](https://pentest-standard.readthedocs.io/en/latest/reporting.html)

Сценарій з Nicolas:

> “external & internal penetration test … only provided the company's name and a network connection onsite … no additional detail.”

– це textbook‑ситуація **black box**: пентестер стартує з практично нульовою інформацією, окрім назви й доступу до мережі, і має самостійно виконати recon, enumeration, побудувати attack paths. [redscan](https://www.redscan.com/news/types-of-pen-testing-white-box-black-box-and-everything-in-between/)

***

### 3. Чому це важливо для репортингу

- Тип assessment’а (VA vs PT) визначає **обсяг та глибину звіту**:
  - VA → фокус на **переліку вразливостей** і remediation.  
  - PT → опис **ланцюгів атак**, доказів експлуатації, бізнес‑імпакту. [aqua-cloud](https://aqua-cloud.io/penetration-testing-report-templates/)

- Перспектива (black/grey/white) визначає **контекст виконаної роботи**:
  - Black box → важливий опис того, що тестер **не знав** спочатку;  
  - Grey/white → треба чітко зафіксувати надану інформацію (доки, креденшли) як частину scope / methodology. [hackmosphere](https://www.hackmosphere.fr/en/penetration-testing-black-box-white-box-and-gray-box/)

Все це потім відображається в розділах звіту: scope, methodology, approach, limitations, threat model.

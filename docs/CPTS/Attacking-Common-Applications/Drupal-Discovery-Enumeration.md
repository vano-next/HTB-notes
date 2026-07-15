# Attacking Common Applications — Drupal: Version Fingerprinting

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Drupal — Discovery & Enumeration](https://academy.hackthebox.com/app/module/113/section/1089)
Ціль: 10.129.42.195 (drupal-qa.inlanefreight.local)
Відповідь: `7.30`

## Концепція

Drupal, як і Joomla/WordPress, розкриває свою версію кількома способами одразу: через мета-тег `Generator` у HTML, через HTTP-заголовок `X-Generator`, і найточніше — через публічний файл `CHANGELOG.txt`, який завжди лежить у корені інсталяції і містить повну історію релізів, де найновіший запис на початку файлу відповідає поточній встановленій версії.

## Крок 1 — Реєстрація vhost і базовий fingerprint

```bash
echo '10.129.42.195 drupal-qa.inlanefreight.local' | sudo tee -a /etc/hosts

curl -s http://drupal-qa.inlanefreight.local | grep -i Drupal
curl -s -I http://drupal-qa.inlanefreight.local
```

Пояснення: `echo ... | sudo tee -a /etc/hosts` дописує рядок з IP-адресою та доменним іменем у кінець файлу `/etc/hosts` (`-a` = append, а не перезапис), щоб операційна система резолвила `drupal-qa.inlanefreight.local` на IP лабораторії локально, без потреби в реальному DNS-сервері — це стандартний перший крок для будь-якого vhost-based таргету в HTB.

Далі виконано два паралельні запити для fingerprint:
- `curl | grep -i Drupal` — витягує рядки з HTML-відповіді, що містять слово "Drupal" (без урахування регістру завдяки `-i`), і одразу знайшов мета-тег:
  ```html
  <meta name="Generator" content="Drupal 7 (http://drupal.org)" />
  ```
- `curl -I` (тільки заголовки, без тіла відповіді) підтвердив те саме через HTTP-заголовок:
  ```
  X-Generator: Drupal 7 (http://drupal.org)
  ```

Обидва джерела узгоджено підтвердили **основну (мажорну) версію — Drupal 7**, але жодне з них не дає точного мінорного номера (наприклад, 7.30 vs 7.58) — сайт свідомо приховує деталізовану версію в цих місцях з міркувань безпеки, тому потрібен точніший метод.

## Крок 2 — Точна версія через CHANGELOG.txt

```bash
curl -s http://drupal-qa.inlanefreight.local/CHANGELOG.txt | grep -m2 ""
```

Пояснення: `CHANGELOG.txt` — стандартний файл Drupal-ядра, який залишається доступним у веброуті практично на кожній інсталяції (рідко хто його видаляє чи блокує). Прапорець `grep -m2 ""` виводить перші 2 рядки файлу незалежно від вмісту (порожній паттерн `""` збігається з будь-яким рядком), що дозволяє швидко побачити заголовок найновішого запису — а оскільки Changelog впорядкований від найновішого релізу до найстарішого, перший заголовок і є точною встановленою версією:

```
Drupal 7.30, 2014-07-24
```

**Відповідь: `7.30`**

## Чому це працює (root cause)

- **Опис:** Drupal за замовчуванням залишає у веброуті кілька джерел, що розкривають версію ядра: HTML meta-тег, HTTP-заголовок `X-Generator` та публічний файл `CHANGELOG.txt` — жодне з них не потребує автентифікації для перегляду.
- **Технічні деталі:** Це не CVE, а стандартна конфігураційна поведінка CMS "з коробки" (CWE-200, Information Exposure Through Discrepancy) — тег generator і файл change­log призначені для розробників/адміністраторів, але ніхто не обмежує до них доступ за замовчуванням.
- **Ризик/Імпакт:** Точна версія 7.30 (реліз 2014 року) дозволяє зловмиснику миттєво звузити пошук публічних CVE — наприклад, знаменита SQL-injection "Drupalgeddon" (CVE-2014-3704) вражає саме версії до 7.32, тож ця версія напряму вказує на конкретний вектор атаки для наступного кроку уроку.
- **Рекомендації:** Видалити або заблокувати доступ до `CHANGELOG.txt`, `README.txt`, `INSTALL.txt` через правила вебсервера; прибрати/підмінити мета-тег `Generator` і заголовок `X-Generator` (модулі типу "Remove Generator Meta Tag"); найважливіше — оновити ядро Drupal до останньої безпечної версії гілки 7.x або мігрувати на підтримувану мажорну версію.

## Швидкий playbook на майбутнє

```bash
# vhost
echo 'IP drupal-qa.inlanefreight.local' | sudo tee -a /etc/hosts

# базовий fingerprint (мажорна версія)
curl -s http://TARGET | grep -i Drupal
curl -s -I http://TARGET | grep -i X-Generator

# точна версія (мінорний номер)
curl -s http://TARGET/CHANGELOG.txt | grep -m2 ""
```

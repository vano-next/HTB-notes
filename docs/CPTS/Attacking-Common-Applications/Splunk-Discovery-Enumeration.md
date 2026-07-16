# Attacking Common Applications — Splunk: Discovery & Enumeration

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)
Урок: [Splunk — Discovery & Enumeration](https://academy.hackthebox.com/app/module/113/section/1092)
Ціль: `10.129.201.50`

Відповідь: `8.2.2`

## Концепція

Splunk надає REST API на порту `8089` (`splunkd`), який за замовчуванням відкриває деякі read-only endpoint'и без автентифікації, включно з `/services/server/info` — це класичний приклад надлишкового розкриття інформації (CWE-200), оскільки версія, ім'я хоста, ОС та роль сервера стають доступні будь-кому без креденшелів. Веб-інтерфейс на порту `8000` натомість вимагає логін і не показує версію в чистому вигляді в HTML, тому спроба через `grep -i version` на сторінці логіну дала лише JS-код кешбастингу (`buildBump`), а не реальний номер продукту.

## Крок 1 — Запит до REST API server/info без автентифікації

```bash
curl -sk https://10.129.201.50:8089/services/server/info
```

Пояснення параметрів: `-k` вимикає перевірку TLS-сертифіката (Splunk за замовчуванням використовує самопідписаний сертифікат на `8089`), `-s` прибирає прогрес-бар curl.

## Крок 2 — Розбір XML-відповіді (Atom feed)

У виводі шукаємо ключ `version` у секції `<s:dict>`:

```xml
<s:key name="version">8.2.2</s:key>
```

Додаткові корисні поля з того ж відповіді для розвідки:
- `build` — `87344edfcdb4` (build hash, корисно для точного визначення патч-рівня)
- `os_name` — `Windows` (ціль на Windows Server, не Linux)
- `host` / `serverName` — `APP03`
- `server_roles` — `indexer`, `license_master` (роль у Splunk-кластері)
- `licenseKeys` — маскований (`FFFF...`), не дає реальних даних
- `guid` — `04EE819F-A5DF-4C6E-B410-661617FCCED8` (унікальний ID інстансу)

**Відповідь: `8.2.2`**

## Чому спроба через порт 8000 не дала результату

```bash
curl -sk https://10.129.201.50:8000/en-US/account/login | grep -i version
```

Це повернуло лише рядок JS-коду:
```
document.write(script({src: baseRoute('/i18ncatalog?autoload=1' + '&version=' + buildBump)}));
```

Пояснення: `buildBump` — це техніка "cache busting" (примусове оновлення кешу браузера при новому білді статичних файлів), а не номер версії Splunk продукту — слово "version" тут стосується версії JS/CSS asset-бандла, а не самого сервера. Веб-інтерфейс приховує реальну версію продукту в HTML до логіну навмисно, тому REST API на `8089` залишається найшвидшим unauthenticated-вектором enumeration.

## Чому це важливо для подальшої атаки

Версія `8.2.2` дозволяє швидко перевірити відомі CVE для Splunk цього релізу (наприклад, вразливості в обробці шаблонів чи ACL bypass у Splunk Web), а факт, що ОС хоста — Windows (а не Linux, як у Tomcat/Jenkins цілях цього модуля), одразу підказує, що подальші кроки експлуатації (RCE через Splunk Enterprise app deployment або через відомі path traversal у деяких версіях) вимагатимуть Windows-специфічних payload'ів замість `/bin/sh`.

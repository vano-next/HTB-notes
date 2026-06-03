# VHost Fuzzing

## Модуль
[Attacking Web Applications with Ffuf — VHost Fuzzing](https://academy.hackthebox.com/app/module/54/section/502)

## Концепція

**VHost (Virtual Host) Fuzzing** — перебір заголовка `Host:` для виявлення прихованих
віртуальних хостів на одному IP. Відрізняється від Sub-domain Fuzzing тим, що:
- DNS-резолюція **не потрібна** — запит іде напряму на IP
- Сервер роутить за заголовком `Host:`
- Знаходить хости, які **не публічно оголошені в DNS**

## Команда

```bash
# 1. Базовий запит — визначити розмір дефолтної відповіді (false positive size)
curl -s http://154.57.164.61:31332/ | wc -c
# → 986 байт — це дефолтна відповідь, яку треба фільтрувати

# 2. VHost fuzzing з фільтрацією за розміром
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
  -u http://154.57.164.61:31332/ \
  -H 'Host: FUZZ.academy.htb' \
  -fs 986
```

### Пояснення ключових параметрів

| Параметр | Що робить |
|----------|-----------|
| `-H 'Host: FUZZ.academy.htb'` | Підставляє кожне слово зі списку як VHost |
| `-fs 986` | Фільтрує відповіді розміром 986 байт (дефолтна сторінка) |
| `-u http://IP/` | Запит іде напряму на IP, без DNS |

## Результат
test [Status: 200, Size: 0]
admin [Status: 200, Size: 0]

**Знайдені VHost-и:**
- `test.academy.htb`
- `admin.academy.htb`

## Різниця: Sub-domain vs VHost Fuzzing

| | Sub-domain Fuzzing | VHost Fuzzing |
|---|---|---|
| DNS | Потрібен | Не потрібен |
| Заголовок | URL (`FUZZ.domain.com`) | `Host: FUZZ.domain.com` |
| Знаходить | Публічні піддомени | Приховані internal vhosts |
| Команда | `-u https://FUZZ.domain.com/` | `-H 'Host: FUZZ.domain.com'` |

## OPSEC / Нотатки

- Завжди визначай `false positive size` перед запуском (`curl | wc -c`)
- Можна фільтрувати також за кодом (`-fc 302`) або словами (`-fw`)
- Для production: додай знайдені хости в `/etc/hosts` для подальшого тестування:
  ```bash
  echo "154.57.164.61  test.academy.htb admin.academy.htb" | sudo tee -a /etc/hosts
  ```

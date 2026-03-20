# External Recon and Enumeration Principles

## Що шукаємо?

| Дані | Опис |
|------|------|
| **IP Space** | ASN організації, netblocks, cloud-провайдери, DNS-записи |
| **Domain Info** | Піддомени, поштові сервери, DNS, VPN-портали, тип захисту (SIEM, AV, IPS/IDS) |
| **Schema Format** | Email-формат, AD username format, password policy — для password spraying |
| **Data Disclosures** | Публічні файли (.pdf, .docx, .xlsx) з внутрішніми посиланнями, метаданими |
| **Breach Data** | Витоки credentials (usernames, passwords) у публічному доступі |

---

## Де шукаємо?

| Ресурс | Інструменти / Сайти |
|--------|--------------------|
| **ASN / IP** | [IANA](https://www.iana.org/), [ARIN](https://www.arin.net/), [RIPE](https://www.ripe.net/), [BGP Toolkit](https://bgp.he.net/) |
| **Domain / DNS** | [Domaintools](https://www.domaintools.com/), [viewdns.info](https://viewdns.info/), [ICANN](https://lookup.icann.org/), `nslookup`, `dig` |
| **Social Media** | LinkedIn, Twitter, Facebook, новинні статті |
| **Сайт компанії** | Сторінки About Us / Contact Us, вбудовані документи |
| **Cloud / Dev** | [GitHub](https://github.com/), [GrayHatWarfare](https://grayhatwarfare.com/), [Google Dorks](https://www.exploit-db.com/google-hacking-database) |
| **Breach Data** | [HaveIBeenPwned](https://haveibeenpwned.com/), [Dehashed](https://www.dehashed.com/) |

---

## Пошук ASN / IP

```bash
# BGP Toolkit (Hurricane Electric) — шукаємо за доменом або IP
https://bgp.he.net/

# Reverse IP lookup — які домени на одному IP
https://viewdns.info/reverseip/?host=TARGET_DOMAIN&t=1
```

---

## DNS Enumeration

```bash
# Отримати всі TXT-записи домену (там може бути флаг / чутлива інфа)
nslookup -type=TXT TARGET_DOMAIN

# Альтернатива через dig
dig TXT TARGET_DOMAIN

# Отримати всі типи записів
dig any TARGET_DOMAIN

# Перевірити nameservers
nslookup ns1.TARGET_DOMAIN
nslookup ns2.TARGET_DOMAIN

# Reverse DNS lookup
nslookup IP_ADDRESS
```

---

## Google Dorks

```bash
# Пошук PDF-файлів на сайті
filetype:pdf inurl:TARGET_DOMAIN

# Пошук email-адрес на сайті
intext:"@TARGET_DOMAIN" inurl:TARGET_DOMAIN

# Пошук внутрішніх посилань / intranet
inurl:TARGET_DOMAIN intext:"intranet"

# Пошук будь-яких документів
filetype:docx OR filetype:xlsx OR filetype:pptx inurl:TARGET_DOMAIN
```

---

## Username Harvesting

```bash
# linkedin2username — скрапимо LinkedIn, генеруємо username-списки
git clone https://github.com/initstring/linkedin2username
cd linkedin2username
pip3 install -r requirements.txt
python3 linkedin2username.py -u YOUR_LI_EMAIL -c COMPANY_NAME
```

Формати username, які генерує інструмент:
- `flast` (перша літера імені + прізвище)
- `first.last`
- `f.last`
- `firstlast`

---

## Credential Hunting (Dehashed)

```bash
# Пошук через скрипт
git clone https://github.com/sm00v/Dehashed
cd Dehashed
pip3 install -r requirements.txt

# Пошук за доменом
python3 dehashed.py -q TARGET_DOMAIN -p
```

Приклад виводу:
```
id      : 5996447501
email   : user@target.local
username: uname
password: P@ssw0rd123!
name    : User Name
```

> Знайдені паролі варто спробувати на зовнішніх порталах: Citrix, OWA, VPN, RDS, VMware Horizon

---

## Trufflehog — пошук секретів у GitHub

```bash
# Встановлення
pip3 install truffleHog

# Сканування репозиторію
trufflehog git https://github.com/TARGET_ORG/TARGET_REPO

# Або через docker
docker run --rm trufflesecurity/trufflehog git https://github.com/TARGET_ORG/TARGET_REPO
```

---

## Практичний приклад: inlanefreight.com

### Крок 1: BGP / ASN перевірка
```
https://bgp.he.net/dns/inlanefreight.com

Результат:
- IP: 134.209.24.248
- MX: mail1.inlanefreight.com
- NS: ns1.inlanefreight.com, ns2.inlanefreight.com
```

### Крок 2: Viewdns — перевірка IP
```
https://viewdns.info/reverseip/?host=inlanefreight.com
```

### Крок 3: Перевірка nameservers
```bash
nslookup ns1.inlanefreight.com
# -> 178.128.39.165

nslookup ns2.inlanefreight.com
# -> 206.189.119.186
```

### Крок 4: TXT-записи (там може бути флаг!)
```bash
nslookup -type=TXT inlanefreight.com
# Виводить TXT-записи, включно з HTB{...}
```

### Крок 5: Google Dorks
```bash
# Пошук PDF
filetype:pdf inurl:inlanefreight.com

# Пошук email-адрес
intext:"@inlanefreight.com" inurl:inlanefreight.com
```

### Крок 6: Breach Data
```bash
python3 dehashed.py -q inlanefreight.local -p
```

---

## Принципи Enumeration

1. **Пасивна розвідка** перед активною — спочатку OSINT, потім сканування
2. **Від широкого до вузького** — починаємо з загальної картини, звужуємося
3. **Ітеративний процес** — повертаємося до recon на кожному етапі тесту
4. **Документуємо все** — скріншоти, файли, вивід команд зберігаємо одразу
5. **Перевіряємо scope** — перед будь-якою активною дією переконуємося, що target в scope

---

## Питання розділу

**Q1:** While looking at inlanefreights public records; A flag can be seen. Find the flag and submit it.

```bash
nslookup -type=TXT inlanefreight.com
# Відповідь: HTB{5Fz6UPNUFFzqjdg0AzXyxCjMZ}
```

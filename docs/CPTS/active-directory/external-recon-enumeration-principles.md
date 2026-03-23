# External Recon and Enumeration Principles

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1264](https://academy.hackthebox.com/app/module/143/section/1264) |

---

## 🎯 Мета

Зовнішня розвідка (External Recon) — збір публічної інформації **без активного сканування**:
- Валідувати scope
- Знайти витоки credentials / документів
- Визначити email-формат та username schema
- Знайти субдомени та сервіси

---

## 🗂️ Що шукаємо

| Data Point | Опис |
|------------|------|
| `IP Space` | ASN, netblocks, cloud provider, DNS записи |
| `Domain Information` | Субдомени, mail servers, VPN portals, захисні системи |
| `Schema Format` | Email формат, AD username формат, password policy |
| `Data Disclosures` | Публічні PDF/DOCX/XLSX з метаданими або інтранет-посиланнями |
| `Breach Data` | Злиті логіни та паролі у відкритому доступі |

---

## 🔧 Ресурси та інструменти

### ASN / IP

| Ресурс | URL |
|--------|-----|
| BGP Toolkit (Hurricane Electric) | https://bgp.he.net/ |
| ARIN (Америка) | https://www.arin.net/ |
| RIPE (Європа) | https://www.ripe.net/ |

### Domain / DNS

| Ресурс | URL |
|--------|-----|
| ViewDNS.info | https://viewdns.info/ |
| DomainTools / Whois | https://whois.domaintools.com/ |
| PTRArchive | http://ptrarchive.com/ |
| ICANN Lookup | https://lookup.icann.org/lookup |

### Breach Data

| Ресурс | URL |
|--------|-----|
| HaveIBeenPwned | https://haveibeenpwned.com/ |
| Dehashed | https://www.dehashed.com/ |

### Cloud / Dev Storage

| Ресурс | URL |
|--------|-----|
| GitHub | https://github.com/ |
| GrayHat Warfare (S3/Azure) | https://grayhatwarfare.com/ |
| Google Hacking DB | https://www.exploit-db.com/google-hacking-database |

---

## 🔑 Ключові команди

### DNS Enumeration

```bash
# TXT записи (там може бути флаг!)
nslookup -type=TXT inlanefreight.com
dig TXT inlanefreight.com
dig ANY inlanefreight.com @8.8.8.8

# Nameservers
nslookup ns1.inlanefreight.com
nslookup ns2.inlanefreight.com

# MX записи
nslookup -type=MX inlanefreight.com
```

### Google Dorks

```bash
# Пошук PDF файлів
filetype:pdf inurl:inlanefreight.com

# Email адреси
intext:"@inlanefreight.com" inurl:inlanefreight.com

# Субдомени
site:*.inlanefreight.com
```

### Dehashed (API)

```bash
sudo python3 dehashed.py -q inlanefreight.local -p
```

### LinkedIn2Username

```bash
python3 linkedin2username.py -c "Inlanefreight" -n 5
```

### Trufflehog (GitHub leak detection)

```bash
trufflehog github --org=inlanefreight --only-verified
```

---

## 🗺️ Процес External Recon (покроково)

```text
1. BGP.he.net → ASN, IP блоки, NS, MX
2. ViewDNS.info → Reverse IP, Whois, DNS report
3. nslookup / dig → валідація IP, TXT записи, MX, NS
4. Google Dorks → PDF/DOCX документи, email адреси
5. LinkedIn → структура компанії, email формат (first.last)
6. HaveIBeenPwned / Dehashed → breach data
7. GitHub / GrayHatWarfare → витоки коду та credentials
```

---

## ❓ Questions

### Q1 — Flag у публічних DNS-записах

> **While looking at inlanefreights public records; A flag can be seen. Find the flag and submit it. (format == HTB{******})**

**Steps:**

```bash
# Виконати DNS TXT запит до inlanefreight.com
nslookup -type=TXT inlanefreight.com

# Або через dig:
dig TXT inlanefreight.com
```

**Очікуваний вивід:**
```
inlanefreight.com    text = "HTB{5Fz6UPNUFFzqjdg0AzXyxCjMZ}"
```

**Answer:** `HTB{5Fz6UPNUFFzqjdg0AzXyxCjMZ}`

> 💡 **Логіка:** TXT-записи в DNS часто містять верифікаційні токени (SPF, Google, Atlassian тощо). Під час зовнішньої розвідки перевірка TXT-записів — стандартний крок, який може виявити чутливу інформацію.

---


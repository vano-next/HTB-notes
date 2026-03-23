# Scenario — Active Directory Enumeration & Attacks

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1263](https://academy.hackthebox.com/app/module/143/section/1263) |

---

## 🎯 Контекст

Ми — пентестери компанії **CAT-5 Security**. Завдання від Team Lead (J. Smith):
- Initial recon та enum домену `INLANEFREIGHT.LOCAL`
- Credential discovery (OSINT + мережева enum)
- Lateral movement та enum внутрішніх сервісів
- Privilege Escalation → Domain Admin

Фінальна ціль: **два** internal pentest проти Inlanefreight.

---

## 🗺️ Assessment Scope

### In Scope

| Range/Domain | Опис |
|-------------|------|
| `INLANEFREIGHT.LOCAL` | Основний домен (AD + web) |
| `LOGISTICS.INLANEFREIGHT.LOCAL` | Субдомен |
| `FREIGHTLOGISTICS.LOCAL` | Дочірня компанія, external forest trust з INLANEFREIGHT.LOCAL |
| `172.16.5.0/23` | Internal subnet |

### Out of Scope

- Будь-які інші субдомени `INLANEFREIGHT.LOCAL` або `FREIGHTLOGISTICS.LOCAL`
- Phishing / Social Engineering
- Будь-які атаки на реальний сайт `inlanefreight.com`
- IP/домени не зазначені в scope

---

## 📋 Методи (дозволені)

### Passive External Recon
- OSINT з анонімної позиції в інтернеті
- Без активного сканування зовнішніх IP
- Без атак на `https://www.inlanefreight.com`

### Internal Testing
- Стартуємо з **анонімної позиції** у внутрішній мережі
- Мета: отримати domain user creds → enum → foothold → lateral/vertical movement → DA
- Не переривати роботу систем під час тесту

### Password Testing
- Захоплені хеші можна крекати офлайн
- Дані зберігаються тільки на схвалених системах CAT-5

---

## 🔑 Key Info

| Параметр | Значення |
|----------|---------|
| Клієнт | Inlanefreight |
| Компанія-тестер | CAT-5 Security |
| Домен | INLANEFREIGHT.LOCAL |
| Forest trust | FREIGHTLOGISTICS.LOCAL (external) |
| Internal subnet | 172.16.5.0/23 |

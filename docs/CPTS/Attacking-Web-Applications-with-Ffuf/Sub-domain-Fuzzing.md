# Sub-domain Fuzzing

**Модуль:** HTB Academy — Attacking Web Applications with FFUF  
**Секція:** Sub-domain Fuzzing  
**Тип:** Web / DNS Enumeration  
**Складність:** Easy  

***

## Огляд

Sub-domain fuzzing — перебір піддоменів шляхом підстановки слів у DNS-ім'я хоста. Дозволяє виявити приховані портали, staging-середовища, адмін-панелі та інші сервіси, не пов'язані з основним сайтом.

***

## Хід виконання

### Команда

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
  -u https://FUZZ.inlanefreight.com/ \
  -t 10
```

| Прапор | Опис |
|--------|------|
| `-w` | Wordlist з топ-5000 субдоменів |
| `FUZZ` | Підставляється замість субдомену у URL |
| `-t 10` | 10 потоків — обережно з публічними доменами |

### Результат

```
customer    [Status: 200, Size: ...]
```

***

## Відповідь

```
customer.inlanefreight.com
```

***

## Нотатки

- `FUZZ` ставиться на місці субдомену: `https://FUZZ.target.com/` — не плутати з directory fuzzing де FUZZ в кінці шляху.
- Для публічних доменів тримай `-t` низьким (10–20), щоб не генерувати зайвий шум.
- Якщо багато false positives (однакові розміри відповідей) — додай фільтр:
  ```bash
  -fs <розмір>   # фільтр за розміром відповіді
  -fc 301,302    # фільтр за кодом редиректу
  ```
- Альтернативні wordlists для ширшого покриття:
  ```
  /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
  /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
  ```
- Для внутрішніх середовищ (без реального DNS) — використовуй vhost fuzzing з `-H "Host: FUZZ.target.com"`.

***

## Sub-domain vs VHost Fuzzing

| | Sub-domain Fuzzing | VHost Fuzzing |
|---|---|---|
| Метод | DNS-запит через URL | HTTP заголовок `Host:` |
| Команда | `https://FUZZ.target.com/` | `-u http://target.com/ -H "Host: FUZZ.target.com"` |
| Застосування | Публічні домени з реальним DNS | Внутрішні / віртуальні хости |
| Фільтрація | За статус-кодом | За розміром відповіді (`-fs`) |

***

## Інструменти

- **ffuf** — основний інструмент для sub-domain fuzzing
- **Seclists** — `Discovery/DNS/subdomains-top1million-5000.txt`
- **amass / subfinder** — пасивна розвідка субдоменів (OSINT)

# ColdFusion — Discovery & Enumeration

Модуль: [Attacking Common Applications](https://academy.hackthebox.com/app/module/113)  
Урок: [ColdFusion - Discovery & Enumeration](https://academy.hackthebox.com/app/module/113/section/2134)  
Ціль: `10.129.107.223`  
Відповідь на Q1: `Server Monitor`

## Крок 1 — Скануємо порти

Починаємо з повного сканування цілі, щоб побачити, які сервіси взагалі доступні:

```bash
nmap -p- -sC -Pn 10.129.107.223 --open
```

У твоєму випадку результат показав:
- `135/tcp open msrpc`
- `8500/tcp open fmtp`
- `49154/tcp open unknown`

Ключова зачіпка тут — **порт 8500**, бо це один із типових портів ColdFusion. ColdFusion часто використовує `8500` як вебпорт серверної конфігурації, тому це хороший перший індикатор технології. [guides.adobe](https://guides.adobe.com/content/coldfusion-docs/en/docs/install-and-configure-coldfusion/administering-coldfusion.html)

## Крок 2 — Відкриваємо вебінтерфейс

Якщо бачимо відкритий `8500`, відкриваємо в браузері:

```text
http://10.129.107.223:8500/
```

Що шукаємо на сторінці:
- директорії типу `CFIDE/`
- директорії типу `cfdocs/`
- файли з розширенням `.cfm`
- адмінку на кшталт:
  ```text
  /CFIDE/administrator/index.cfm
  ```

Це типові сліди ColdFusion під час enumeration. Якщо бачиш `CFIDE` або `administrator`, це майже пряме підтвердження, що ціль працює на ColdFusion. [0xss0rz.gitbook](https://0xss0rz.gitbook.io/0xss0rz/pentest/web-attacks/coldfusion)

## Крок 3 — Фіксуємо ознаки ColdFusion

Під час перегляду сайту звертаємо увагу на типові артефакти:
- URL із `.cfm` або `.cfc`
- директорію `CFIDE`
- сторінку `ColdFusion Administrator`
- повідомлення про помилки, де згадується ColdFusion

Це важливо не лише для відповіді на питання, а й для майбутньої експлуатації: такі ознаки допомагають зрозуміти версію продукту, набір стандартних шляхів і можливі вектори атаки. [0xss0rz.gitbook](https://0xss0rz.gitbook.io/0xss0rz/pentest/web-attacks/coldfusion)

## Крок 4 — Відповідь на питання

Питання: **What ColdFusion protocol runs on port 5500?**  
Правильна відповідь: **`Server Monitor`**. У документації та технічних описах ColdFusion порт `5500` пов’язаний саме з моніторингом сервера. [carehart](https://www.carehart.org/blog/2012/2/24/cf901_enable_server_monitoring_myth)

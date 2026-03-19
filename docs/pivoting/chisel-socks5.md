# SOCKS5 Tunneling with Chisel

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1437](https://academy.hackthebox.com/module/158/section/1437) |

---

## 🔑 Key Concepts

- **Chisel** — fast TCP/UDP tunnel over HTTP, secured via SSH
- Server runs on **pivot host**, client runs on **attack host**
- Creates SOCKS5 proxy on `127.0.0.1:1080` for proxychains

### Flow
```
Kali (chisel client) ——→ Pivot Host (chisel server:1234) ——→ Internal Network
proxychains:1080 ————————————————————————————————————————→ 172.16.5.x
```

---

## 🛠️ Key Commands

### Build or Download Chisel (Kali)
```bash
# Build from source
git clone https://github.com/jpillora/chisel.git
cd chisel && go build

# Or download prebuilt binary
wget https://github.com/jpillora/chisel/releases/latest/download/chisel_linux_amd64.gz
gunzip chisel_linux_amd64.gz && mv chisel_linux_amd64 chisel && chmod +x chisel
```

### Transfer Chisel to Pivot Host
```bash
scp chisel ubuntu@<pivot_host>:~/
```

### Start Chisel Server on Pivot Host
```bash
ssh ubuntu@<pivot_host>   # pass: HTB_@cademy_stdnt!
chmod +x chisel
./chisel server -v -p 1234 --socks5
# Should show: server: Listening on http://0.0.0.0:1234
```

### Start Chisel Client on Kali
```bash
./chisel client -v <pivot_host>:1234 socks
# Should show: client: Connected (Latency X ms)
# Listens on 127.0.0.1:1080
```

### Verify Tunnel
```bash
ss -tlnp | grep 1080
# Should show: chisel listening on 127.0.0.1:1080
```

### Configure Proxychains for SOCKS5
```bash
sudo bash -c 'cat > /etc/proxychains.conf << EOF
strict_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5 127.0.0.1 1080
EOF'
```

### RDP via Chisel Tunnel
```bash
proxychains xfreerdp3 /v:172.16.5.19 /u:victor /p:pass@123 /cert:ignore
```

---

## ❓ Questions

### Q1 — Contents of C:\Users\victor\Documents\flag.txt
**Answer:** `Th3$eTunne1$@rent8oring!`

```bash
# TERMINAL 1 — Kali: transfer chisel
scp chisel ubuntu@10.129.23.176:~/

# TERMINAL 2 — Pivot host: start server
ssh ubuntu@10.129.23.176
./chisel server -v -p 1234 --socks5

# TERMINAL 3 — Kali: start client
./chisel client -v 10.129.23.176:1234 socks

# TERMINAL 4 — Kali: RDP to internal target
proxychains xfreerdp3 /v:172.16.5.19 /u:victor /p:pass@123 /cert:ignore
# CMD: type C:\Users\victor\Documents\flag.txt
```

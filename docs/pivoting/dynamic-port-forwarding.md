# Dynamic Port Forwarding with SSH and SOCKS Tunneling

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1426](https://academy.hackthebox.com/module/158/section/1426) |

---

## 🔑 Key Commands

### SSH Dynamic Port Forwarding
```bash
ssh -D 9050 -N ubuntu@<pivot_host>
```

### Setup Proxychains
```bash
sudo bash -c 'cat > /etc/proxychains.conf << EOF
strict_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks4 127.0.0.1 9050
EOF'
```

### Scan via Proxychains
```bash
proxychains nmap -v -Pn -sT -p<port> <target_ip>
```

### RDP via Proxychains
```bash
proxychains xfreerdp3 /v:<target> /u:<user> /p:<pass> /cert:ignore
```

### Verify Tunnel is Active
```bash
ss -tlnp | grep 9050
```

---

## ❓ Questions

### Q1 — How many NICs on pivot host?
**Answer:** `3` (ens192, ens224, lo)

```bash
ssh ubuntu@10.129.202.64   # pass: HTB_@cademy_stdnt!
ifconfig
```

### Q2 — Flag.txt via RDP Pivot
**Answer:** `N1c3Piv0t`

```bash
# TERMINAL 1 — keep open
ssh -D 9050 -N ubuntu@10.129.202.64

# TERMINAL 2
proxychains nmap -v -Pn -sT -p3389 172.16.5.19
proxychains xfreerdp3 /v:172.16.5.19 /u:victor /p:pass@123 /cert:ignore
```

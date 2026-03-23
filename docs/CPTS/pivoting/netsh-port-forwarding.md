# Port Forwarding with Windows: Netsh

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1435](https://academy.hackthebox.com/module/158/section/1435) |

---

## 🔑 Key Concepts

- **Netsh** — Windows built-in tool for network configuration
- `portproxy` — forwards traffic from one port/IP to another
- Useful when pivot host is Windows and no third-party tools are available

### Flow
```
Kali → Windows Pivot:8080 → netsh portproxy → 172.16.5.19:3389
```

---

## 🛠️ Key Commands

### RDP to Windows Pivot Host
```bash
xfreerdp3 /v:<pivot_ip> /u:htb-student /p:HTB_@cademy_stdnt! /cert:ignore
```

### Add Port Forwarding Rule (CMD as Admin on Windows)
```cmd
netsh.exe interface portproxy add v4tov4 ^
  listenport=8080 ^
  listenaddress=<pivot_ip> ^
  connectport=3389 ^
  connectaddress=172.16.5.19
```

### Verify Rule
```cmd
netsh.exe interface portproxy show v4tov4
```

### Allow Port in Windows Firewall
```cmd
netsh advfirewall firewall add rule name="Open Port 8080" ^
  protocol=TCP dir=in localport=8080 action=allow
```

### Connect to Internal Target via Pivot
```bash
xfreerdp3 /v:<pivot_ip>:8080 /u:victor /p:pass@123 /cert:ignore
```

### Delete Rule (Cleanup)
```cmd
netsh.exe interface portproxy del v4tov4 listenport=8080 listenaddress=<pivot_ip>
```

---

## ❓ Questions

### Q1 — Approved contact name in VendorContacts.txt?
**Answer:** `Jim Flipflop`

```bash
# Step 1 — RDP to Windows pivot
xfreerdp3 /v:10.129.42.198 /u:htb-student /p:HTB_@cademy_stdnt! /cert:ignore

# Step 2 — CMD on Windows pivot (add port forward)
netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.42.198 connectport=3389 connectaddress=172.16.5.19

# Step 3 — From Kali, RDP to internal target via pivot
xfreerdp3 /v:10.129.42.198:8080 /u:victor /p:pass@123 /cert:ignore

# Step 4 — Open file
# Desktop → Approved Vendors → VendorContacts.txt → Jim Flipflop
```

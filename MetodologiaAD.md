# Mi Metodología — Active Directory

Esto no es una referencia de comandos (eso vive en `AD-Guia-Completa.md`). Esto es **cómo pienso** al atacar un set de AD, construido a partir de lo que ya probé que sé hacer solo en 7 máquinas reales (Active, Forest, Sauna, Monteverde, Timelapse, Return, Fluffy). Cuando me trabe en el examen, vuelvo a ESTE documento primero — es el árbol de decisiones, no el libro de comandos.

---

## PASO 0 — Orientarme

```
nmap/rustscan → ¿veo 88+389+445+5985 juntos? → es AD → DC a /etc/hosts
```
Lo primero que determino: **¿tengo credenciales ya (breach scenario), o empiezo desde cero?**
- Si HTB/examen me dio un usuario:password desde el inicio → reviso el "About"/enunciado SIEMPRE antes de escanear nada (lección de Fluffy: perdí tiempo buscando lo que ya me habían dado).
- Si no → empiezo en el Paso 1.

---

## PASO 1 — Enumeración sin credenciales: busco DOS cosas en paralelo

No enumero "por enumerar" — busco específicamente **(A) usuarios** y **(B) credenciales filtradas**, porque son los dos únicos resultados que me permiten avanzar.

**(A) Usuarios** — pruebo null session, RPC anónimo, LDAP anónimo, y si hay puerto 80, la web:
```
nxc smb -u '' -p ''  /  rpcclient -U "" -N  /  ldapsearch anónimo  /  revisar la web
```
Regla que ya aprendí: **si una fuente da pocos o ningún usuario, cruzo con otra antes de asumir que la lista está completa** (en Forest, LDAP anónimo escondía usuarios que RPC sí mostraba).

**(B) Credenciales filtradas** — reviso TODOS los shares accesibles, no solo los "normales":
```
smbclient -L  →  entro a cada share con -N  →  ls/recurse  →  bajo TODO lo sospechoso
```
Qué busco específicamente en lo que descargo:
- `Groups.xml` → GPP, `cpassword` cifrado (clave pública conocida)
- `.xml` con `PSADPasswordCredential` → password en texto plano (patrón Azure AD Connect)
- `.zip`/`.pfx` → cracking en cascada (zip2john/pfx2john + john)
- `.pdf`/`.docx` con nombres de CVEs → pista directa de un exploit específico que debo buscar
- Cualquier `.txt`/notas → leídas completas, no solo escaneadas

**Si tengo puerto 80 y nada de lo anterior da fruto:** reviso la web en busca de nombres de empleados → `username-anarchy` → valido contra el dominio con `nxc`.

**Decisión al final del Paso 1:** ¿tengo una credencial real? Si sí → Paso 3. Si no → Paso 2.

---

## PASO 2 — Si el Paso 1 no dio una credencial directa

Antes de rendirme, pruebo en este orden (son rápidos, no cuestan mucho tiempo):
1. Password spraying con lista vacía y comunes: `nxc smb -u users.txt -p ''`
2. **Password spraying usuario=password** (truco real que me funcionó en Monteverde: `-u users.txt -p users.txt`)
3. AS-REP Roasting contra toda mi lista de usuarios (no necesita credencial previa)

Si ESTO tampoco da nada, reviso si hay una pista de CVE específica en algo que descargué (patrón Fluffy) — eso significa que el vector no es "credencial encontrada" sino "explotación activa" (capturar un hash vía archivo malicioso, por ejemplo).

---

## PASO 3 — Ya tengo una credencial. Antes de nada: ¿dónde funciona y qué me dice?

```bash
nxc smb <SUBNET>/24 -u '<user>' -p '<pass>'
```
Busco **`(Pwn3d!)`** — es información gratis sobre dónde ya soy admin local, antes de hacer cualquier otra cosa.

Luego enumero con esta credencial (usuarios, grupos, shares — ahora con más acceso que anónimo) y **corro BloodHound de inmediato**. No pospongo BloodHound — es mi mapa, lo necesito antes de decidir cualquier siguiente paso.

---

## PASO 4 — Leer el mapa de BloodHound: mi orden de prioridad real

Ya until to aprendí que el número de saltos en el grafo ENGAÑA. Mi orden real, de más confiable a menos:

```
1º — ¿Tengo un ACL directo o por cadena corta hacia algo útil?
     GenericAll / WriteDacl / GetChanges+GetChangesAll sobre el dominio
     → lo ejecuto YA, sin depender de nada más
     (si es cadena de grupos: net rpc → verificar con members → si falla
      silenciosamente, ldapmodify o bloodyAD como alternativa)

2º — ¿GenericWrite/GenericAll sobre una CUENTA específica (no un grupo)?
     → Shadow Credentials (certipy shadow auto) en vez de resetear password
     → me da el hash NT directo, sin tocar su contraseña real

3º — ¿Kerberoastable o AS-REP roasteable?
     → lo pido YA, pero sé que depende de que el hash crackee
     (nombre tipo SVC_* = pienso Kerberoast primero, no AS-REP)

4º — ¿Solo veo HasSession/AdminTo hacia una máquina que NO controlo?
     → lo dejo de último, esconde trabajo previo que el grafo no muestra
```

**Si NINGUNO de estos aparece en BloodHound**, no asumo que estoy trabado — reviso estas 3 rutas alternativas que ya sé que existen:
```
¿Grupo "Azure Admins" o cuentas AAD_*?        → Azure AD Connect / ADSync decrypt
¿Puedo correr nxc ldap -M laps?               → LAPS, password de Administrator local
¿Hay CA de certificados en el dominio?        → certipy find -vulnerable → ESC16/otro ESC
```

---

## PASO 5 — Conseguí un nuevo acceso. ¿Shell o solo datos?

Si lo que gané es una **shell nueva** (WinRM, psexec, wmiexec):
```
Confirmo puerto ANTES de elegir herramienta (5985 vs 5986=SSL vs 445)
WinRM conecta pero no ejecuta nada → no es error mío, revisar grupo
  "Remote Management Users" → cambio a psexec/wmiexec
```

Una vez dentro, mi checklist de post-explotación en orden:
```
1. whoami /priv  y  whoami /groups     → privilegios especiales (SeBackupPrivilege, etc.)
2. dir C:\Users\<yo>\Desktop           → mi propia flag, si la hay
3. ConsoleHost_history.txt             → credenciales que alguien tecleó antes
4. net user /domain                    → lista de usuarios (y confirmo que es /domain, no local)
5. Si nada obvio: winPEAS              → servir desde la carpeta correcta, IP de VPN,
                                          convertir UTF-16→UTF-8 antes de grep
```

Si lo que gané es solo un **hash/credencial nueva** (sin shell directa): vuelvo al Paso 3 con esa credencial — repito BloodHound, repito el Paso 4.

---

## PASO 6 — Llegar al final: DCSync, LAPS, o certificado — y las flags

```
DCSync confirmado (edge o recién concedido)  → secretsdump -just-dc → hash de Administrator
LAPS                                          → password de Administrador LOCAL de esa máquina
AD CS / ESC16                                 → cambiar UPN → pedir cert → certipy auth → hash
```

Con el hash/password de Administrator: Pass-the-Hash o login directo.

**Última lección que no debo olvidar:** Administrator lee **cualquier** carpeta del sistema — si la flag está en `C:\Users\OtroUsuario\Desktop`, no necesito "ser" ese usuario, solo necesito ser Administrator y navegar ahí.

Si un comando da "Access denied" a pesar de ser Administrator y `whoami /priv` muestra algo como `SeBackupPrivilege` → el privilegio no se activa solo, necesito la herramienta correcta (`robocopy /B`).

---

## Mis 3 reglas de oro para cuando algo "no funciona"

Estas son las que más se repitieron en mis propias sesiones — las reviso ANTES de pedir ayuda:

1. **¿2+ protocolos distintos (SMB, LDAP, WinRM, Kerberos) rechazan la misma credencial?** → el dato está mal copiado/tecleado, no la técnica. Lo reescribo a mano.
2. **¿Un comando "no dio error" pero tampoco parece haber hecho nada?** → verifico el resultado explícitamente (ej: `net rpc group members` después de un `addmem`). Nunca asumo éxito por ausencia de error.
3. **¿Una herramienta de Python se rompe con un traceback de `cryptography`/`aioquic`?** → es un conflicto de dependencias de algo que instalé antes, no mi comando. Venv aislado o `pip uninstall cryptography --break-system-packages -y`.

---

## Cuándo SÍ pido ayuda (y no me siento mal por eso)

Mirando mis propias sesiones, el patrón es claro: pido ayuda cuando me topo con una **técnica que nunca he visto**, no cuando me trabo en el flujo general. Eso es correcto y debo seguir así:
- La primera vez que algo aparece (Shadow Credentials, ESC16, SeBackupPrivilege, Azure AD Connect decrypt) — es razonable no saberlo de memoria.
- Lo que ya NO debería requerir ayuda: sintaxis de comandos que ya usé antes, diagnóstico de errores de red/VPN, instalación de herramientas que ya instalé una vez, confusión Kerberoast/AS-REP (ya la tengo clara).

La métrica que me importa no es "cuánta ayuda pedí", es **"¿la ayuda que pedí fue por algo genuinamente nuevo, o por algo que ya debería saber resolver solo?"**

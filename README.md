# OSCP-Notes

Notas personales de preparación para el examen OSCP / OSCP+. Referencia offline para usar durante la práctica y el examen (`Ctrl+F` por puerto, técnica o herramienta).


## Índice

| # | Archivo | Contenido |
|---|---------|-----------|
| 0 | [0-Reglas-Examen.md](0-Reglas-Examen.md) | Formato, puntos, herramientas permitidas/prohibidas |
| 1 | [1-Recon.md](1-Recon.md) | Escaneo de puertos y enumeración web |
| 2 | [2-Servicios.md](2-Servicios.md) | Enumeración por servicio (SMB, FTP, LDAP, MSSQL, etc.) |
| 3 | [3-Explotacion.md](3-Explotacion.md) | Explotación web (SQLi, LFI, upload…) + exploits + transferencia |
| 4 | [4-Shells.md](4-Shells.md) | Reverse shells Linux/Windows, TTY, msfvenom |
| 5 | [5-PrivEsc.md](5-PrivEsc.md) | Escalada de privilegios Linux y Windows |
| 6 | [6-AD.md](6-AD.md) | **Active Directory (40 pts) — árbol de decisión: IF puerto/situación → vectores** |
| 6b | [6b-AD-Referencia-Detallada.md](6b-AD-Referencia-Detallada.md) | AD — referencia de comandos detallada (el libro de comandos con todos los flags) |
| 7 | [7-Pivoting.md](7-Pivoting.md) | Túneles y pivoting (ligolo, chisel, SSH) |
| 8 | [8-Cracking.md](8-Cracking.md) | Password cracking (hashcat, John, Hydra) |
| — | [MetodologiaAD.md](MetodologiaAD.md) | AD — "cómo pienso" (metodología en prosa) |
| — | [ExamPlan.md](ExamPlan.md) | Metodología, árboles de decisión y template de reporte |
| — | [9-Tracking-Maquinas.md](9-Tracking-Maquinas.md) | Seguimiento de máquinas practicadas |
| — | [writeups/](writeups/) | Writeups detallados de máquinas reales (cadenas completas + lecciones) |
| — | [cheatsheet/](cheatsheet/) | Cheatsheets y study notes en HTML (abrir en el navegador): [AD de inicio a fin](cheatsheet/ad-study-notes.html), [cheatsheet ampliada](cheatsheet/oscp-cheatsheet-ampliada.html), [walkthrough Access](cheatsheet/access-walkthrough.html) |

## La sección de AD, en dos niveles

1. **[6-AD.md](6-AD.md) — el árbol de decisión.** El "qué hacer cuando veo X", estructurado como IFs:
   - **IF por puerto** → "veo 88+389+445 → es un DC → P1"; tabla puerto → significado → vectores.
   - **IF por posición (9 posiciones)** → "tengo 1 credencial → BloodHound, Kerberoast, LAPS, shares…"; cada botín te reubica.
   - Con comandos listos para copiar y lecciones reales (`🔴`) sacadas de los writeups.
2. **[6b-AD-Referencia-Detallada.md](6b-AD-Referencia-Detallada.md) — el libro de comandos.** Cuando el árbol te manda a una técnica y necesitas todos los flags, casos raros y gotchas largos.

## Writeups de máquinas reales

Cadenas completas resueltas paso a paso, con el *por qué* de cada decisión y las lecciones para el examen:

| Máquina | Cadena (resumen) |
|---|---|
| [heist](writeups/heist-writeup.md) | SSRF → NetNTLMv2 → WinRM → gMSA → SeRestorePrivilege → SYSTEM |
| [resourced](writeups/resourced-writeup.md) | ntds.dit en share → volcado offline → hash-spray → WinRM → RBCD → DA |
| [vault-gpo](writeups/vault-gpo-writeup.md) | NTLM theft (`.url`) → crackeo → WinRM → GPO abuse (Default Domain Policy) → DA |
| [feast](writeups/feast-writeup.md) | S3 anónimo → webshell SYSTEM → creds MySQL → ForceChangePassword → LSA secrets → OPtH → DA |
| [hutch](writeups/hutch-laps-writeup.md) | LDAP anon → pass en `description` → LAPS read (`ms-Mcs-AdmPwd`) → admin del DC |
| [astronaut-grav](writeups/astronaut-grav-writeup.md) | Grav CMS CVE-2021-21425 (YAML write) → RCE → SUID php → root (Linux) |
| [internal-server2008](writeups/internal-server2008-writeup.md) | SMB → MS09-050 (CVE-2009-3103) → SYSTEM (standalone, no dominio) |
| [kali-python-viejo-uv](writeups/kali-python-viejo-uv.md) | Correr PoC con Python viejo en Kali rolling usando `uv` |

## Flujo rápido

```
Recon (1) → Enum servicios (2) → Explotación (3) → Shell + TTY (4)
   → PrivEsc (5) → [si es dominio] AD (6 / 6b) + Pivoting (7)
   → Cracking (8) cuando aparezcan hashes
AD: "¿qué hago cuando veo X?" → 6-AD.md (árbol de decisión)
Metodología y "qué hacer cuando encuentro X": ExamPlan.md / MetodologiaAD.md
```

## Uso

Clónalo **localmente antes del examen** — no dependas de que GitHub cargue durante las 24h:

```bash
git clone https://github.com/jfabio-navarrom/OSCP-Notes.git
```

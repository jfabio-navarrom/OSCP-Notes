# Writeups — máquinas resueltas (referencia reproducible)

Writeups detallados de máquinas de práctica (HTB / Proving Grounds). Cada uno documenta la **cadena completa y reproducible** más las **lecciones** y los callejones sin salida, que es lo que de verdad se memoriza.

> Úsalos de dos formas: (1) como plantilla mental del flujo "enum → foothold → privesc", y (2) `Ctrl+F` por técnica (RBCD, gMSA, SeRestore, GPO abuse…) cuando te topes con algo parecido en el examen. Las técnicas sueltas están además condensadas en los cheat sheets numerados (`6-AD.md`, `5-PrivEsc.md`, etc.).

## Índice

| Máquina | OS / Tipo | Cadena resumida | Técnicas clave |
|---------|-----------|-----------------|----------------|
| [Heist](heist-writeup.md) | Windows / DC (dominio) | SSRF → NetNTLMv2 → gMSA → SeRestore → SYSTEM | SSRF forzado, Responder, gMSA read, `SeRestorePrivilege` (Utilman swap) |
| [ResourceD](resourced-writeup.md) | Windows / DC (dominio) | Backup ntds.dit → hash-spray → RBCD → SYSTEM | `secretsdump` offline, hash-spray, **RBCD** (S4U) |
| [Vault](vault-gpo-writeup.md) | Windows / DC (dominio) | Share escribible → NTLM theft → crack → GPO abuse → DA | NTLM theft (`.url` IconFile), **GPO abuse** (pyGPOAbuse) |
| [Hutch](hutch-laps-writeup.md) | Windows / DC (dominio) | LDAP anon → pass en `description` → LAPS → SYSTEM | Hunting de descripciones LDAP, **LAPS** (`ms-Mcs-AdmPwd`) |
| [Astronaut](astronaut-grav-writeup.md) | Linux (Grav CMS) | CVE-2021-21425 (YAML write) → www-data → SUID php → root | Localizar la ruta de la app, SUID intérprete (GTFOBins) |
| [Internal](internal-server2008-writeup.md) | Windows Server 2008 (workgroup) | SMB vuln → MS09-050 → SYSTEM | `nmap --script smb-vuln*`, mapa CVE↔boletín MS |

## Notas de entorno

| Nota | Para qué |
|------|----------|
| [kali-python-viejo-uv.md](kali-python-viejo-uv.md) | Correr PoC viejos con Python 3.9/3.11 vía `uv` (Kali rolling solo trae 3.14) |

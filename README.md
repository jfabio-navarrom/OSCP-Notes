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
| 6 | [6-AD.md](6-AD.md) | Active Directory (40 pts) |
| 7 | [7-Pivoting.md](7-Pivoting.md) | Túneles y pivoting (ligolo, chisel, SSH) |
| 8 | [8-Cracking.md](8-Cracking.md) | Password cracking (hashcat, John, Hydra) |
| — | [ExamPlan.md](ExamPlan.md) | Metodología, árboles de decisión y template de reporte |
| — | [MetodologiaAD.md](MetodologiaAD.md) | Cómo pensar un set de AD (árbol de decisiones, no comandos) |
| — | [9-Tracking-Maquinas.md](9-Tracking-Maquinas.md) | Hoja de tracking de práctica (foco en independencia) |
| — | [writeups/](writeups/) | Writeups reproducibles de máquinas resueltas + técnicas clave |

## Flujo rápido

```
Recon (1) → Enum servicios (2) → Explotación (3) → Shell + TTY (4)
   → PrivEsc (5) → [si es dominio] AD (6) + Pivoting (7)
   → Cracking (8) cuando aparezcan hashes
Metodología y "qué hacer cuando encuentro X": ExamPlan.md
AD, cómo pensarlo: MetodologiaAD.md   |   Ejemplos reales: writeups/
```

## Writeups de máquinas

En [`writeups/`](writeups/) hay cadenas completas y reproducibles de máquinas ya resueltas (Heist, ResourceD, Vault, Hutch, Astronaut, Internal), cada una con sus lecciones y callejones sin salida. Las técnicas que aparecen ahí (RBCD, GPO abuse, gMSA, LAPS, SeRestore, NTLM theft, SUID intérprete…) están además condensadas en los cheat sheets numerados para buscarlas rápido con `Ctrl+F`.

## Uso

Clónalo **localmente antes del examen** — no dependas de que GitHub cargue durante las 24h:

```bash
git clone https://github.com/jfabio-navarrom/OSCP-Notes.git
```

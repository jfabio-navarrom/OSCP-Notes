# Hoja de Tracking de Máquinas — OSCP

Registro de práctica enfocado en **independencia**. La métrica que importa no es cuántas máquinas rooteaste, sino cuántas rooteaste **100% solo** (sin writeup, sin video, sin IA, sin pista de foro).

> Cómo llenarla: una fila por máquina. En **Indep.** pon ✅ solo si la rooteaste de principio a fin sin abrir ninguna ayuda. Si usaste cualquier pista → ❌ (aunque la hayas terminado). Anota SIEMPRE dónde te trabaste y qué aprendiste — esa columna es tu activo más valioso.

---

## Tablero de progreso (los gates del plan)

Marca cuando cumplas cada hito. No se negocian: son requisitos para avanzar de fase.

- [ ] **Gate Fase 1 → 2:** 5 máquinas **Easy** resueltas 100% solas
- [ ] **Gate Fase 2 → 3:** 10–15 máquinas solas acumuladas (Easy+Medium) **y** 3 Medium seguidas sin pista
- [ ] **Gate Fase 3 → 4:** 1 set de **AD (3 máquinas)** de punta a punta solo
- [ ] **Listo para examen:** 1 simulacro de ≥70 pts con reporte entregable

**Contadores (actualízalos a mano):**

| Métrica | Cuenta |
|---------|--------|
| Máquinas intentadas (total) | 2 |
| Resueltas 100% solas ✅ | 0 |
| Resueltas con pista ❌ | 2 |
| **Tasa de independencia** (solas ÷ total) | 0% |
| Medium seguidas sin pista (racha actual) | 0 |
| Sets de AD completados solo | 0 |

> Objetivo de tasa de independencia antes del examen: **≥70%** y subiendo. Si a mitad de Fase 2 sigue baja, estás abriendo writeups demasiado rápido — aguanta más el "trabarse productivo". **Nota real de hoy:** 0% no es fracaso — es tu línea base honesta. Lo que importa es que la tendencia suba conforme avances, no el número de hoy.

---

## FASE 1 — Easy (destete) · regla: 1 h solo antes de mirar

| # | Máquina | Fuente | OS | Fecha | Tiempo | Indep. | ¿Dónde me trabé? | Lección (→ al cheat sheet) |
|---|---------|--------|----|-------|--------|--------|------------------|----------------------------|
| 1 | Cap | HTB | Linux | 2026-09 | — | ✅/❌ | *(completar)* | *(completar)* |
| 2 |  |  |  |  |  |  |  |  |
| 3 |  |  |  |  |  |  |  |  |
| 4 |  |  |  |  |  |  |  |  |
| 5 |  |  |  |  |  |  |  |  |
| 6 |  |  |  |  |  |  |  |  |
| 7 |  |  |  |  |  |  |  |  |
| 8 |  |  |  |  |  |  |  |  |
| 9 |  |  |  |  |  |  |  |  |
| 10 |  |  |  |  |  |  |  |  |
| 11 |  |  |  |  |  |  |  |  |
| 12 |  |  |  |  |  |  |  |  |

---

## FASE 2 — Easy→Medium (independencia) · regla: 2 h solo + re-enum ×2, solo título del paso

| # | Máquina | Fuente | OS | Dific. | Fecha | Tiempo | Indep. | ¿Dónde me trabé? | Lección |
|---|---------|--------|----|--------|-------|--------|--------|------------------|---------|
| 1 | Active | HTB | Windows/AD | Easy | 2026-09-25 | ~2.5h (con ayuda) | ❌ | (1) Confundí `GetNPUsers` con `GetUserSPNs` — no diferenciaba AS-REP de Kerberoasting. (2) Usé evil-winrm sin confirmar antes que 5985 estaba abierto → ECONNREFUSED, perdí tiempo. (3) Instalación de BloodHound CE desde cero (espacio en disco, collation de Postgres) consumió la mayor parte del tiempo, no la técnica en sí. | Ver `6-AD.md` actualizado: sección "NO CONFUNDIR" de Kerberos, y "elige herramienta según puerto abierto". Patrón completo: GPP/Groups.xml → gpp-decrypt → creds → BloodHound → Kerberoasting → hashcat → PtH/creds directas al DC. |
| 2 |  |  |  |  |  |  |  |  |  |
| 3 |  |  |  |  |  |  |  |  |  |
| 4 |  |  |  |  |  |  |  |  |  |
| 5 |  |  |  |  |  |  |  |  |  |
| 6 |  |  |  |  |  |  |  |  |  |
| 7 |  |  |  |  |  |  |  |  |  |
| 8 |  |  |  |  |  |  |  |  |  |
| 9 |  |  |  |  |  |  |  |  |  |
| 10 |  |  |  |  |  |  |  |  |  |
| 11 |  |  |  |  |  |  |  |  |  |
| 12 |  |  |  |  |  |  |  |  |  |
| 13 |  |  |  |  |  |  |  |  |  |
| 14 |  |  |  |  |  |  |  |  |  |
| 15 |  |  |  |  |  |  |  |  |  |
| 16 |  |  |  |  |  |  |  |  |  |
| 17 |  |  |  |  |  |  |  |  |  |
| 18 |  |  |  |  |  |  |  |  |  |
| 19 |  |  |  |  |  |  |  |  |  |
| 20 |  |  |  |  |  |  |  |  |  |
| 21 |  |  |  |  |  |  |  |  |  |
| 22 |  |  |  |  |  |  |  |  |  |
| 23 |  |  |  |  |  |  |  |  |  |
| 24 |  |  |  |  |  |  |  |  |  |
| 25 |  |  |  |  |  |  |  |  |  |
| 26 |  |  |  |  |  |  |  |  |  |
| 27 |  |  |  |  |  |  |  |  |  |
| 28 |  |  |  |  |  |  |  |  |  |
| 29 |  |  |  |  |  |  |  |  |  |
| 30 |  |  |  |  |  |  |  |  |  |

---

## FASE 3 — AD + cadenas + pivoting · regla: 3 h solo, solo 1 línea del writeup

| # | Máquina / Set | Fuente | Foco (AD/chain/pivot) | Fecha | Tiempo | Indep. | ¿Dónde me trabé? | Lección |
|---|---------------|--------|-----------------------|-------|--------|--------|------------------|---------|
| 1 |  |  | AD |  |  | ✅/❌ |  |  |
| 2 |  |  |  |  |  |  |  |  |
| 3 |  |  |  |  |  |  |  |  |
| 4 |  |  |  |  |  |  |  |  |
| 5 |  |  |  |  |  |  |  |  |
| 6 |  |  |  |  |  |  |  |  |
| 7 |  |  |  |  |  |  |  |  |
| 8 |  | Set AD (3 máq.) |  |  |  |  |  |  |

---

## FASE 4 — Simulacros de examen · regla: 0 pistas, 0 IA, cronómetro real

| # | Simulacro | Máquinas | Puntos | ¿Reporte escrito? | Duración | Notas / debilidades detectadas |
|---|-----------|----------|--------|-------------------|----------|-------------------------------|
| 1 |  | 3 stand + 1 AD |  /100 | Sí/No |  |  |
| 2 |  |  |  /100 | Sí/No |  |  |
| 3 |  |  |  /100 | Sí/No |  |  |

---

## Registro de rabbit holes (para no repetirlos)

Cada vez que caigas en uno, anótalo. El patrón que repites es tu debilidad real.

| Máquina | En qué me perdí | Cuánto tiempo perdí | Cómo salí | Señal para la próxima |
|---------|-----------------|---------------------|-----------|-----------------------|
| Active | Instalación de BloodHound CE desde cero (espacio en disco lleno, error de collation de PostgreSQL, confusión Neo4j vs interfaz real en :8080) | ~1h+ | Expandí partición con growpart/resize2fs, refresqué collation con `ALTER DATABASE ... REFRESH COLLATION VERSION` | **Ya resuelto para siempre** — BloodHound queda instalado y configurado; la próxima máquina de AD no debería tener este obstáculo. Verificar espacio en disco (`df -h`) al inicio de cada sesión nueva de todos modos. |
| Active | Probé `evil-winrm` sin confirmar antes si el puerto 5985 estaba abierto | ~5 min | Vi `ECONNREFUSED`, cambié a `impacket-wmiexec` sobre el 445 | Antes de elegir herramienta de conexión (evil-winrm vs psexec/wmiexec), correr `nmap -p445,5985,5986 <IP>` primero — no asumir. |
| Active | Confundí `GetNPUsers` (AS-REP) con `GetUserSPNs` (Kerberoasting) — intenté correr el comando equivocado para el ataque que quería hacer | ~10 min | Corregido en el momento revisando `6-AD.md` | Regla mental ya anotada en el repo: si el nombre de cuenta suena a cuenta de servicio (`SVC_*`), pensar primero en Kerberoasting, no AS-REP. |

---

## Debilidades recurrentes (revísalo cada semana)

Si la misma categoría aparece mucho en "¿dónde me trabé?", ahí va tu tiempo de estudio.

- [ ] Enumeración incompleta (puertos/servicios que salté)
- [ ] Web (SQLi / LFI / upload)
- [ ] PrivEsc Linux
- [ ] PrivEsc Windows
- [x] Active Directory — confusión entre técnicas de Kerberos (AS-REP vs Kerberoasting); confirmar puerto antes de elegir herramienta de conexión
- [ ] Pivoting
- [ ] Cracking / hashes
- [ ] Gestión de tiempo / rabbit holes
- [x] Setup de entorno/herramientas — instalación de BloodHound consumió tiempo desproporcionado (resuelto, no debería repetirse)

---

*Recuerda: 20 máquinas con writeup valen menos que 10 solas. Cuenta las solas. Lista de máquinas: https://docs.google.com/spreadsheets/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/edit*

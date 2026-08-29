# OSCP — Metodología, Toma de Decisiones y Template de Reporte

> Compañero del cheat sheet. Aquí no hay comandos nuevos: hay **cómo pensar**. El 80% de los fracasos en OSCP no son por falta de un comando, son por no saber qué hacer con lo que ya encontraste o por meterse en un rabbit hole y no salir.

---

## 1. El loop mental (la única "metodología" que importa)

```
ENUMERAR → encontrar algo → preguntarme "¿esto para qué me sirve?" →
probar la hipótesis más simple primero → si funciona, ¿qué me da ahora? →
si no, VOLVER A ENUMERAR (más profundo) — nunca saltar de máquina al azar
```

Tres preguntas que te haces ante CUALQUIER hallazgo:
1. **¿Qué es?** (versión exacta, rol, para qué sirve)
2. **¿Me da acceso o me da información?** (¿es un vector de entrada o una pista?)
3. **¿Dónde lo reutilizo?** (credenciales, nombres de usuario, rutas → sirven en OTRAS máquinas/servicios)

**Regla de oro:** cada credencial, usuario, hash, o ruta que encuentres se prueba en **todos** los servicios y **todas** las máquinas. La reutilización de credenciales es el vector #1 que la gente pasa por alto.

---

## 2. Árbol de decisiones: "cuando encuentro X → hago Y"

### 🔹 Encuentro un puerto abierto raro / no estándar
- Banner grab: `nc TARGET PUERTO`, `curl`, `telnet`.
- Busca la versión exacta → `searchsploit servicio version`.
- Si es HTTP en puerto raro (8080, 8000, 8443…) → trátalo como web completa.
- Si no lo reconoces → googléalo (en prep, no en examen): "puerto XXXX default service".
- **No lo ignores por ser "raro".** Los puertos no estándar suelen ser la vía intencional.

### 🔹 Encuentro una web
1. Código fuente (Ctrl+U), comentarios HTML, `robots.txt`, `sitemap.xml`.
2. `whatweb` / headers → ¿qué CMS/framework/versión?
3. Fuzzing de directorios Y de archivos con extensiones (`.php`, `.bak`, `.txt`, `.zip`).
4. ¿Hay login? → default creds, SQLi, enumeración de usuarios.
5. ¿Hay parámetros GET/POST? → SQLi, LFI, command injection, SSTI.
6. ¿Hay upload? → webshell.
7. ¿CMS conocido con versión? → `searchsploit`.
8. ¿Subdominios/vhosts? → añade a `/etc/hosts` y fuzz de nuevo.

### 🔹 Encuentro credenciales (user:pass) — el hallazgo más valioso
Pruébalas en **TODO**, en este orden:
- SSH, WinRM (`evil-winrm`), RDP.
- SMB (`nxc smb`), y valida contra toda la subred si es dominio.
- El panel web / la DB / el servicio donde las hallaste y los demás.
- ¿Es una máquina de dominio? → úsalas para BloodHound + Kerberoast.
- Variaciones: la misma pass con otros usuarios, la pass + año/`!`/`123`.

### 🔹 Encuentro un hash
1. Identifícalo: `hashid` / `hash-identifier`.
2. ¿NTLM? → intenta **Pass-the-Hash** antes de crackear (no siempre necesitas la clave).
3. Crackéalo: hashcat con el modo correcto (ver tabla del cheat sheet) + rockyou.
4. Si crackea → trátalo como credencial (árbol de arriba).

### 🔹 Encuentro un nombre de usuario
- Constrúyete una lista `users.txt` con TODOS los usuarios que veas (web, SMB, comentarios, LDAP).
- Úsala para: AS-REP roasting, password spraying, brute force dirigido.
- Prueba convenciones de nombres (jsmith, j.smith, smithj) si es dominio.

### 🔹 Consigo un foothold (shell de usuario bajo) — NO celebres aún
1. Estabiliza la shell (TTY).
2. **Captura `local.txt` YA** + screenshot con IP + `whoami`/`hostname`.
3. Enumeración de privesc: `sudo -l` (Linux) / `whoami /priv` (Windows) SIEMPRE primero.
4. Corre linpeas/winpeas Y revisa manualmente.
5. Busca credenciales en: configs, historial, `/var/www`, DBs, `.ssh`, archivos de backup.
6. Antes de kernel exploits → agota SUID/servicios/cron/sudo/capabilities.

### 🔹 Consigo root/SYSTEM/Admin
1. **Captura `proof.txt` YA** + screenshot completa.
2. Dump de credenciales (`secretsdump`, mimikatz, `/etc/shadow`, SAM).
3. Esas credenciales → pívot a las OTRAS máquinas (sobre todo en AD).
4. Documenta el paso final reproducible antes de pasar a otra máquina.

### 🔹 Veo puerto 88 (Kerberos) / 389 (LDAP) / 445 en varias máquinas
- Es Active Directory. Ve al plan de AD del cheat sheet.
- Cualquier credencial de dominio, por baja que sea → BloodHound primero (te dibuja el camino a Domain Admin).

### 🔹 Encuentro servicios internos (127.0.0.1 / puertos que no salían en el escaneo externo)
- `netstat -tulnp` (Linux) / `netstat -ano` (Windows) desde tu foothold.
- Hay algo escuchando solo en local → **pivoting** (ligolo/chisel) para alcanzarlo.
- Suele ser el vector de la siguiente máquina o de privesc.

### 🔹 Encuentro un exploit público (searchsploit / GitHub)
1. **Léelo antes de correrlo.** Entiende qué hace (en el examen no puedes usar auto-exploit tools, pero exploits manuales sí).
2. Ajusta IP/puerto/offsets a tu caso.
3. Ten un listener listo si es un RCE/reverse.
4. Si falla, revisa: ¿versión exacta?, ¿arquitectura?, ¿dependencias?, ¿necesita compilar (`gcc`)?

---

## 3. Manejo de RABBIT HOLES (esto te salva el examen)

Un rabbit hole es un camino que *parece* el correcto pero no lleva a ningún lado. OSCP los pone a propósito.

**Señales de que estás en uno:**
- Llevas >45–60 min en el mismo vector sin progreso.
- Estás forzando un exploit que "debería" funcionar pero no.
- Encontraste algo llamativo (un CVE famoso) y te obsesionaste sin confirmarlo.
- Estás adivinando en vez de seguir evidencia.

**Qué hacer:**
1. **Time-box todo.** Ponte límites: 20 min por hipótesis. Si no avanza, la marcas como "pendiente" y sigues.
2. **Vuelve a enumerar, más profundo.** El 90% de las veces la salida estaba en algo que no leíste: un puerto que no escaneaste, un directorio que no fuzzeaste, output que ignoraste.
3. **Pregúntate: ¿esto es intencional?** Si un servicio es la versión más nueva y parcheada, probablemente NO es el vector.
4. **Lee TODO otra vez.** Banners, comentarios, código fuente, nombres de archivos. La pista suele estar a la vista.
5. **Cambia de capa, no de máquina.** Si el vector web no da → prueba otro servicio de la MISMA máquina antes de abandonarla.
6. **Descansa 10 min.** En serio. La mitad de los rabbit holes se resuelven volviendo con la cabeza fresca.
7. **Anota lo que YA descartaste** para no repetirlo a la hora 15.

**Antipatrón clásico:** encontrar un CVE crítico, tirar exploits una hora, y que el vector real fuera credenciales por defecto en otro puerto. Enumera ancho antes de profundizar.

---

## 4. Errores que hacen fallar (y cómo evitarlos)

| Error | Antídoto |
|-------|----------|
| No escanear los 65535 puertos / saltar UDP | Escaneo completo desde el minuto 1 |
| No hacer `sudo -l` / `whoami /priv` al conseguir shell | Es lo PRIMERO tras estabilizar |
| No reutilizar credenciales entre máquinas | Toda cred/hash se prueba en todo |
| Ir directo a kernel exploits | Última opción; agota lo demás |
| Saltar de máquina cuando te trabas | Vuelve a enumerar la misma |
| No capturar screenshots en el momento | Captura flag apenas la tengas |
| Quemar Metasploit temprano | Guárdalo para 1 sola máquina difícil |
| No documentar el paso reproducible | Anota comando+output al instante |
| Empezar el reporte sin haberlo practicado | Practícalo 2–3 veces antes |
| Burnout a la hora 12 | Come liviano, descansos reales |

---

## 5. Template de Reporte OSCP

> Practícalo 2–3 veces con máquinas de PG Practice ANTES del examen. Los errores en el reporte real son permanentes. OffSec da una plantilla oficial en .docx en el portal — úsala como base; esto es la estructura mental.
>
> **Recordatorio: el reporte NO se escribe con IA. Es fase de examen.**

```
════════════════════════════════════════════════
  OFFENSIVE SECURITY — OSCP EXAM REPORT
  Nombre: [tu nombre]        OSID: OS-XXXXX
  Fecha del examen: [fecha]
════════════════════════════════════════════════

1. INTRODUCCIÓN / RESUMEN EJECUTIVO
   - Alcance del examen (rango de IPs, objetivos).
   - Resumen de resultados: X de Y máquinas comprometidas, Z puntos.

2. METODOLOGÍA / ALCANCE
   - Breve descripción del enfoque (enum → explotación → privesc → post).

3. POR CADA MÁQUINA (repetir el bloque):
────────────────────────────────────────
   TARGET: 10.x.x.x  (Hostname si aplica)
   
   3.1 Enumeración de servicios
       - Output relevante de Nmap (puertos/servicios).
       - Screenshot o texto del escaneo.
   
   3.2 Explotación / Acceso inicial
       - Vulnerabilidad identificada (con detalle técnico).
       - Comando/exploit EXACTO usado (reproducible).
       - Screenshot de la shell obtenida + `whoami`/`id`.
       - local.txt: [contenido]  ← screenshot con IP + hostname visibles.
   
   3.3 Escalada de privilegios
       - Cómo se identificó el vector (sudo -l, SUID, servicio, etc.).
       - Pasos EXACTOS y reproducibles.
       - Screenshot de root/SYSTEM + `whoami`/`id`.
       - proof.txt: [contenido]  ← screenshot con IP + hostname visibles.
   
   3.4 Post-explotación (opcional/AD)
       - Credenciales/hashes obtenidos y cómo se usaron para pivotar.
────────────────────────────────────────

4. (Solo AD) DIAGRAMA / RUTA DE ATAQUE DEL DOMINIO
   - Foothold → creds → lateral movement → DC.

5. REMEDIACIÓN (por hallazgo)
   - Recomendaciones prácticas para cada vulnerabilidad.

6. APÉNDICE
   - Comandos completos, exploits usados, referencias.
```

**Reglas de oro del reporte:**
- **Reproducible = aprobado.** El corrector debe poder repetir tu ataque solo con tu reporte. Si un paso no es reproducible, no cuenta.
- Cada screenshot de flag DEBE mostrar: la IP del target + el hash del flag + `whoami`/`hostname` (o `type proof.txt`) en la MISMA captura.
- No inventes ni rellenes. Solo lo que hiciste y probaste.
- Formato de entrega exacto: PDF → comprimir a `.7z` con el nombre de archivo que indica la guía → subir al portal dentro de las **24h** posteriores al examen.
- Nombra bien las imágenes y numéralas.

---

## 6. Mini-checklist para pegar al lado del monitor

```
AL EMPEZAR
[ ] Nmap -p- en TODAS las máquinas (background)
[ ] UDP top-ports en todas
[ ] Anotar cada puerto → servicio → versión

POR CADA SERVICIO
[ ] Banner + versión + searchsploit
[ ] Default creds
[ ] Enumeración específica (SMB/web/etc.)

AL CONSEGUIR SHELL
[ ] Estabilizar TTY
[ ] CAPTURAR local.txt (IP+whoami en screenshot)
[ ] sudo -l  /  whoami /priv   ← ¡PRIMERO!
[ ] linpeas/winpeas + revisión manual
[ ] Buscar creds en configs/historial/DB

AL CONSEGUIR ROOT/ADMIN
[ ] CAPTURAR proof.txt (IP+whoami en screenshot)
[ ] Dump de credenciales
[ ] Reutilizar creds en otras máquinas (AD)
[ ] Documentar el paso reproducible

SI ME TRABO (>45 min)
[ ] Volver a enumerar MÁS profundo
[ ] ¿Es intencional este vector?
[ ] Otro servicio de la MISMA máquina
[ ] Descanso 10 min

CADA 3-4 HORAS
[ ] ¿Tengo screenshots de todo lo logrado?
[ ] ¿Voy con el puntaje para 70?
[ ] Comer / hidratar / descansar
```

---

*El examen no premia saber más comandos que nadie. Premia enumerar completo, reutilizar todo lo que encuentras, salir rápido de los rabbit holes y documentar de forma reproducible. Eso es toda la metodología.*

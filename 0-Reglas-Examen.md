# 0. Reglas del Examen y Herramientas

Lee esto antes que nada. Usar una herramienta prohibida por descuido puede descalificarte.

## Formato (OSCP+ actual)

- **3 máquinas standalone** → 20 pts c/u = 60 pts
- **1 conjunto de Active Directory** (3 máquinas) → 40 pts
- **70 puntos para aprobar.** Sin buffer overflow. Sin bonus points.
- **23h45** de examen práctico + **24h** para escribir y entregar el reporte.
- Necesitas `local.txt` (usuario) y `proof.txt` (root/admin) de cada máquina.

## Herramientas — RESTRICCIONES

**Metasploit / Meterpreter:**
- Solo en **UNA** máquina en todo el examen (una sola vez).
- No se puede usar para pivoting (contaría como usarlo en más de un target).
- Los módulos auxiliares/scanners de MSF (no exploits) se pueden usar libremente.

**PROHIBIDO por completo:**
- Herramientas de **explotación automática**: SQLMap, SQLNinja, db_autopwn, browser_autopwn, wpscan en modo agresivo/explotación.
- **Escáneres de vulnerabilidades** masivos: Nessus, OpenVAS, Nexpose, Qualys.
- **Spoofing / poisoning**: Responder solo en modo *análisis* (`-A`), nunca envenenamiento (nada de `-w`, LLMNR/NBT-NS poisoning).
- Ataques automáticos de fuerza bruta masiva sobre el propio panel del examen.
- **IA / LLMs / chatbots** (ChatGPT, Claude, Copilot, etc.) durante el examen Y durante el reporte.

**PERMITIDO:**
- Nmap (+ scripts NSE), Nikto, dirb/gobuster/feroxbuster/ffuf, Burp Suite Community.
- Hydra, Medusa, Hashcat, John.
- SQLi **manual** (¡solo SQLMap está prohibido!), exploits públicos manuales.
- Impacket, evil-winrm, NetExec/CrackMapExec, BloodHound, SharpHound.
- Mimikatz, PowerView, Rubeus, PrintSpoofer, GodPotato.
- chisel, ligolo-ng, socat, proxychains.
- Utilidades estándar de línea de comandos.

> Regla mental: si una herramienta **encuentra Y explota sola** una vulnerabilidad, probablemente está prohibida. Si tú controlas cada paso, casi siempre está permitida.

## Pruebas por cada flag (para el reporte)

Cada screenshot de flag debe mostrar, en la MISMA captura:
- La **IP** del target
- El contenido del **flag** (`local.txt` / `proof.txt`)
- `whoami` / `id` / `hostname` (o `type proof.txt` en Windows)

## Estrategia de puntos para llegar a 70

- Camino común: **AD completo (40)** + **2 standalone (40)** = 80, con margen.
- O: **3 standalone (60)** + acceso inicial parcial + parte del AD.
- Empieza por lo que mejor dominas. Muchos arrancan por AD o por 1 standalone de calentamiento.
- Reserva 2–3 h al final solo para screenshots y pruebas faltantes.

## Entrega del reporte

- PDF → comprimir a **.7z** con el nombre de archivo exacto que indica la guía → subir al portal.
- Dentro de las **24h** posteriores al fin del examen. **El reporte es obligatorio** aunque tengas los puntos.
- **Reproducible = aprobado.** Si un paso no se puede repetir solo con tu reporte, no cuenta.

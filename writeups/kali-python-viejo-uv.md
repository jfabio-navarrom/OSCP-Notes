# Cómo correr scripts con un Python viejo en Kali (uv) — guía rápida

**El problema:** Kali rolling solo trae Python nuevo (3.14) y ya no tiene `python3.11` por apt. Muchos PoC/herramientas (salt, exploits viejos, etc.) **no funcionan en 3.14**. Además Kali bloquea `pip install` global ("externally-managed-environment"). La solución es **uv**, que se trae su propio Python aislado y gestiona las dependencias sin venvs a mano.

---

## 0. Instalar uv (ya lo hiciste con el curl)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```
Después, recarga el PATH (usas **zsh**):
```bash
source ~/.zshrc        # o cierra y abre la terminal
uv --version           # debe responder con una versión
```
Si `uv` no se encuentra, añade el bin a mano:
```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

---

## 1. Instalar un Python viejo (lo baja uv, no apt)

```bash
uv python install 3.11      # o 3.9, 3.10, 3.12...
uv python list              # ver las versiones que tienes
```
Esto descarga un Python 3.11 independiente del sistema. No toca tu 3.14 ni Kali.

---

## 2. Correr un script con ese Python + sus dependencias (lo importante)

**Forma 1 — todo en un comando, sin venv manual:**
```bash
# <paquete> es lo que el script necesita (ej: salt, impacket, pycryptodome...)
uv run --python 3.11 --with <paquete> script.py <argumentos>
```
Ejemplo real con el PoC de SaltStack:
```bash
cd CVE-2020-11651-poc
uv run --python 3.11 --with salt exploit.py --master <IP> --exec 'id'
```
`uv run` crea un entorno temporal por debajo, instala `salt` en el Python 3.11 y ejecuta el script. **No editas el script ni peleas con pip.**

**Varios paquetes a la vez:**
```bash
uv run --python 3.11 --with salt --with pycryptodome exploit.py <args>
```

**Desde un requirements.txt:**
```bash
uv run --python 3.11 --with-requirements requirements.txt exploit.py <args>
```

---

## 3. Si prefieres un entorno fijo (venv con uv, para un repo que usas mucho)

```bash
cd el-repo
uv venv --python 3.11 .venv        # crea el venv con Python 3.11
source .venv/bin/activate          # actívalo
uv pip install -r requirements.txt # instala deps (uv pip, rápido)
python exploit.py <args>           # ya corres con 3.11
deactivate                         # salir del venv
```
La diferencia con la Forma 1: aquí el entorno queda guardado en `.venv/` y lo reutilizas; en la Forma 1 es efímero (ideal para algo de una sola vez).

---

## 4. Herramientas CLI (impacket, etc.) → pipx, no uv

```bash
sudo apt install -y pipx && pipx ensurepath
pipx install impacket
pipx list                 # ver lo instalado
pipx upgrade impacket
```
pipx instala cada herramienta en su propio entorno y te deja el comando disponible globalmente (`impacket-secretsdump`, etc.).

---

## 5. Chuleta mental

| Necesito... | Uso |
|-------------|-----|
| Correr un PoC viejo una vez | `uv run --python 3.11 --with <pkg> script.py <args>` |
| Un repo con requirements que uso seguido | `uv venv --python 3.11 .venv` + `uv pip install -r requirements.txt` |
| Una herramienta de línea de comandos (impacket) | `pipx install <tool>` |
| Forzar pip global (último recurso, NO recomendado) | `pip install <pkg> --break-system-packages` |

---

## 6. Reglas para no volver a atascarte

1. **Nunca edites el script del PoC** para "adaptarlo" a tu Python. El problema es la versión de Python, no el script. Usa `uv run --python 3.X`.
2. **Si un PoC dice que necesita Python 2** (muy viejo, ej. algunos AutoBlue): `uv python install 2.7` no existe; usa un contenedor: `docker run -it --rm --network host -v $PWD:/x python:2.7 bash`.
3. **Ten esto montado ANTES del examen**: uv instalado, Python 3.9/3.11 ya descargados (`uv python install 3.9 3.11`), impacket por pipx. Así no instalas nada bajo presión.
4. **El error "externally-managed-environment"** NO es un fallo tuyo: es la política de Kali. La respuesta correcta es uv/pipx/venv, no pelear con pip global.

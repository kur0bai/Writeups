# FindYourStyle

Pequeña Máquina en modo **easy** de Dockerlabs.

- [Reconocimiento](#reconocimiento)
- [Escaneo](#escaneo)
- [Enumeración](#enumeración)
- [Explotación](#explotación)
- [Escalada de privilegios](#escalada-de-privilegios)

<br/>

## Reconocimiento

La máquina objetivo se encuentra correctamente desplegada dentro de la red de laboratorio (en este caso, utilizando Docker).  
Para identificarla se realizó el uso de `arp-scan` para identificar los dispositivos en nuestra red docker con la interfaz `docker0`

```bash
sudo arp-scan -I docker0 --localnet
Interface: docker0, type: EN10MB, MAC: 02:42:77:20:48:b6, IPv4: 172.17.0.1
WARNING: Cannot open MAC/Vendor file ieee-oui.txt: Permission denied
WARNING: Cannot open MAC/Vendor file mac-vendor.txt: Permission denied
Starting arp-scan 1.10.0 with 65536 hosts (https://github.com/royhills/arp-scan)
172.17.0.3	02:46:tr:15:00:02	(Unknown: locally administered)
```

<br />

## Escaneo

Se realizó un escaneo con **Nmap** para identificar puertos abiertos y servicios:

```
> nmap 172.17.0.3 -p- -sV -sC -Pn --min-rate 5000
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-16 16:54 EDT
Nmap scan report for 172.17.0.3
Host is up (0.0000080s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-generator: Drupal 8 (https://www.drupal.org)
|_http-server-header: Apache/2.4.25 (Debian)
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.txt /web.config /admin/
| /comment/reply/ /filter/tips/ /node/add/ /search/ /user/register/
| /user/password/ /user/login/ /user/logout/ /index.php/admin/
|_/index.php/comment/reply/
|_http-title: Welcome to Find your own Style | Find your own Style
MAC Address: 02:42:AC:11:00:03 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.07 seconds


```

Indicando un servicio corriendo en el puerto **80** de **Drupal**, un CMS en su versión 8.

<br/>

## Enumeración

Se realizó una inspección al servidor web desde el navegador, buscando posibles vulnerabilidades, a su vez se determinó que la versión específica de Drupal era la **8.5.0**

![Drupal](https://i.imgur.com/yRbVifq.png)

Se enumeró con **Gobuster**, al encontrar algunos errores se establecieron filtros al momento de ejecutar nuevamente, debido a que arrojó muchos resultados pero se considerarons los códigos de response desde `200` a `300`.

```bash
> gobuster dir -u "172.17.0.3" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,php.bak -t 20 -o gobuster.txt | grep -v "(Status: 403)"
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.3
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
/index.php            (Status: 200) [Size: 8860]
/contact              (Status: 200) [Size: 12134]
/search               (Status: 302) [Size: 360] [--> http://172.17.0.3/search/node]
/user                 (Status: 302) [Size: 356] [--> http://172.17.0.3/user/login]
/themes               (Status: 301) [Size: 309] [--> http://172.17.0.3/themes/]
/modules              (Status: 301) [Size: 310] [--> http://172.17.0.3/modules/]
/node                 (Status: 200) [Size: 8756]
/Search               (Status: 302) [Size: 360] [--> http://172.17.0.3/search/node]
/sites                (Status: 301) [Size: 308] [--> http://172.17.0.3/sites/]
/Contact              (Status: 200) [Size: 12116]
/core                 (Status: 301) [Size: 307] [--> http://172.17.0.3/core/]
/install.php          (Status: 301) [Size: 318] [--> http://172.17.0.3/core/install.php]
/profiles             (Status: 301) [Size: 311] [--> http://172.17.0.3/profiles/]
/README.txt           (Status: 200) [Size: 5889]
/robots.txt           (Status: 200) [Size: 1596]
/LICENSE.txt          (Status: 200) [Size: 18092]
/User                 (Status: 302) [Size: 356] [--> http://172.17.0.3/user/login]
/SEARCH               (Status: 302) [Size: 360] [--> http://172.17.0.3/search/node]
/rebuild.php          (Status: 301) [Size: 318] [--> http://172.17.0.3/core/rebuild.php]
/CONTACT              (Status: 200) [Size: 12116]
/Node                 (Status: 200) [Size: 8756]

Progress: 1102785 / 1102785 (100.00%)
===============================================================
Finished
===============================================================
```

Se encontró un directorio user que indicaba un formulario de 3 pestañas que ejecutaba diferentes acciones para gestionar usuarios y que al parecer tenía algunos detalles de **Insecure Design** ya que se pudo confirmar la existencia en base de datos del usuario.

![Drupal](https://i.imgur.com/H2N9VRt.png)

![Drupal](https://i.imgur.com/8yRf0VG.png)

Se hizo revisión de otros archivos como `robots.txt` y el `install.php` donde se determinó que la versión instalada era la **8.5.0**

![Drupal](https://i.imgur.com/jlTTSxk.png)

Gracias a esto se pudo revisar información un poco más detallada de las **CVES** encontradas para esta versión y al parecer hay un **RCE** que afecta al core y puede ser inyectado desde los formularios, en este caso puede ser desde `/Users` así que se realizaron algunos estudios de **PoC** para replicar en el target con burpsuite.

![Drupal](https://i.imgur.com/e06L0bw.png)

![Drupal](https://i.imgur.com/sSFsjfs.png)

<br />

## Explotación

Por otro lado se incluyó búsqueda de payloads referentes a la **CVE** y versión con metasploit y se encontraron hallazgos interesantes, como el `Druppalgeddon2` que estaba escrito en ruby, sin embargo se realizó una prueba con meterpreter y el resultado fue positivo.

![Metasploit](https://i.imgur.com/TyzBFWL.png)
![Metasploit](https://i.imgur.com/TuPpEbG.png)

Al ejecutar el exploit se consiguió una sessión en meterpreter de esta manera se procede a explotar y conectarse a la máquina.

![Metasploit](https://i.imgur.com/wPNi4Ky.png)

Luego con el comando `whoami` se determinó que se inició la shell con el usuario `www-data` por lo que se buscó usuarios haciendo un `cat /etc/passwd` para obtener posibles usuarios que tengan acceso al `/bin/bash` y el resultado arrojó que por medio de el usuario `ballenita` se podría obtener.

![Ballenita](https://i.imgur.com/guwYW1T.png)

Lo siguiente era encontrar la contraseña y se realizó enumeraciones dentro del sistema de archivos que se consideraran importantes, incluyendo configuraciones, el resultado más acertado fue el de la misma configuración del Drupal.

```bash
find / -name settings.php 2>/dev/null
/var/www/html/sites/default/settings.php
```

![Ballenita](https://i.imgur.com/yPXzIQr.png)

![Ballenita](https://i.imgur.com/p2r1r1N.png)

Con esto se consiguió la contraseña para la configuración de base de datos **mysql** y el usuario `ballenita`.

<br />

## Escalada de privilegios

Lo siguiente fue realizar un `su ballenita` para cambiar de usuario utilizando la contraseña previamente obtenida del archivo de configuración.

```bash
www-data@430d587770d2:/var/www/html$ su ballenita
su ballenita
Password: ballenitafeliz
ballenita@430d587770d2:
```

Lo siguiente fue ejecutar el comando `sudo -l` para revisar qué binarios tenía permitido ejecutar y si no se encontraba ningún resultado se procedería a buscar binarios con SUID, afortunadamente se halló `/bin/ls` y `/bin/grep`, con los cuales se hizo el intento de acceder al directorio de `/root` y el resultado fue positivo, obteniendo la contraseña del mismo.

![Ballenita](https://i.imgur.com/cf2kQdw.png)

De esta manera se consiguió escalar y obtener el control total de la máquina.

_Written by **kur0bai**_

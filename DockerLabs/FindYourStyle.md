# FindYourStyle

<center><img src="https://dockerlabs.es/static/images/logos/logo.png" width="150px"></center>

## Contents

- [Reconnaissance](#reconnaissance)
- [Scanning](#scanning)
- [Enumeration](#enumeration)
- [Exploitation](#exploitation)
- [Privilege Escalation](#privilege-escalation)

## Reconnaissance

The target machine is correctly deployed inside the lab network (in this case, using Docker).  
To identify it, `arp-scan` was used to find devices on our docker network using the `docker0` interface

```bash
sudo arp-scan -I docker0 --localnet
Interface: docker0, type: EN10MB, MAC: 02:42:77:20:48:b6, IPv4: 172.17.0.1
WARNING: Cannot open MAC/Vendor file ieee-oui.txt: Permission denied
WARNING: Cannot open MAC/Vendor file mac-vendor.txt: Permission denied
Starting arp-scan 1.10.0 with 65536 hosts (https://github.com/royhills/arp-scan)
172.17.0.3	02:46:tr:15:00:02	(Unknown: locally administered)
```

## Scanning

A **Nmap** scan was performed to identify open ports and services:

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

Indicating a service running on port **80** using **Drupal**, a CMS in version 8.

## Enumeration

The web server was inspected from a browser looking for potential vulnerabilities, and the specific Drupal version was determined to be **8.5.0**

![Drupal](https://i.imgur.com/yRbVifq.png)

Directory enumeration was run with **Gobuster**; after encountering many results, filters were applied to consider only response codes from `200` to `300`.

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

A `user` directory was found containing a 3-tab form that performed different actions to manage users. It appeared to have **Insecure Design** issues since the existence of the user in the database could be confirmed.

![Drupal](https://i.imgur.com/H2N9VRt.png)

![Drupal](https://i.imgur.com/8yRf0VG.png)

Other files such as `robots.txt` and `install.php` were reviewed and confirmed the installed version was **8.5.0**

![Drupal](https://i.imgur.com/jlTTSxk.png)

Thanks to this, further research into CVEs affecting this version revealed an **RCE** that impacts the core and can be injected from forms — in this case possibly via `/Users` — so PoC tests were prepared to reproduce it against the target using Burp Suite.

![Drupal](https://i.imgur.com/e06L0bw.png)

![Drupal](https://i.imgur.com/sSFsjfs.png)

## Exploitation

Additionally, payload searches for the relevant **CVE** and version were performed in Metasploit, turning up interesting findings such as `Drupalgeddon2` (written in Ruby). A test using Meterpreter produced a positive result.

![Metasploit](https://i.imgur.com/TyzBFWL.png)
![Metasploit](https://i.imgur.com/TuPpEbG.png)

Executing the exploit provided a Meterpreter session, allowing further actions and connection to the machine.

![Metasploit](https://i.imgur.com/wPNi4Ky.png)

Then, using `whoami` it was determined the shell was running as `www-data`, so `/etc/passwd` was inspected to find potential users with `/bin/bash` access — the user `ballenita` was identified as a candidate.

![Ballenita](https://i.imgur.com/guwYW1T.png)

The next step was to find the password. Filesystem enumeration focused on important configurations, and the most relevant result was the Drupal settings file.

```bash
find / -name settings.php 2>/dev/null
/var/www/html/sites/default/settings.php
```

![Ballenita](https://i.imgur.com/yPXzIQr.png)

![Ballenita](https://i.imgur.com/p2r1r1N.png)

This revealed the database MySQL configuration password and the `ballenita` user.

## Privilege Escalation

Next, `su ballenita` was used to switch users with the password obtained from the configuration file.

```bash
www-data@430d587770d2:/var/www/html$ su ballenita
su ballenita
Password: ballenitafeliz
ballenita@430d587770d2:
```

Then `sudo -l` was executed to check allowed binaries; if nothing was allowed, SUID binaries would be searched. Fortunately, `/bin/ls` and `/bin/grep` were found, which were used to access `/root` and retrieve the root password.

![Ballenita](https://i.imgur.com/cf2kQdw.png)

This allowed escalation and full control of the machine.

---

_Written by **kur0bai**_

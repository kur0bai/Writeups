# ConsoleLog

<center><img src="https://dockerlabs.es/static/images/logos/logo.png" width="150px"></center>

## Contents

- [Reconnaissance](#reconnaissance)
- [Scanning](#scanning)
- [Enumeration](#enumeration)
- [Exploitation](#exploitation)
- [Privilege Escalation](#privilege-escalation)

## Reconnaissance

The target machine is deployed inside the lab network (here using Docker). To identify it, `arp-scan` was used to find devices on the `docker0` interface:

```bash
sudo arp-scan -I docker0 --localnet
Interface: docker0, type: EN10MB, MAC: 02:42:77:20:48:b6, IPv4: 172.17.0.1
Starting arp-scan 1.10.0 with 65536 hosts (https://github.com/royhills/arp-scan)
172.17.0.2	02:42:ac:11:00:02	(Unknown: locally administered)
```

## Scanning

An **Nmap** scan was performed to identify open ports and services:

```bash
nmap -p- --open -sC -sV --min-rate 5000 -n -Pn 172.17.0.2
```

Target IP identified: **172.17.0.2**

![Scan1](https://i.imgur.com/1epCh4F.png)

## Enumeration

Next step was searching for hidden directories and resources.

Using **Gobuster**:

```bash
gobuster dir -u "http://172.17.0.2"   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -t 20 -x php,txt,html,php.bak
```

The scan revealed a directory containing frontend JavaScript files and references to an endpoint called `/recurso`.

![Gobuster](https://i.imgur.com/zmVJle6.png)

Inspecting the web application in the browser console revealed a message referring to a recurso directory and a token expected by the backend.

![Console message](https://i.imgur.com/bC2z0KO.png)

Further server inspection showed that the backend directory had **Directory Listing** enabled, allowing direct reading of files from the service root.

![Directory Listing](https://i.imgur.com/QwUb4sa.png)

Among the exposed files, `server.js` was found. It implements a **POST** `/recurso` endpoint.

![server.js](https://i.imgur.com/ZNjSEUB.png)

The endpoint compares a received token with a hardcoded value (tokentraviesito) and, if it matches, returns a password in plaintext: `lapassworddebackupmaschingonadetodas`.

> **Note**: Even without directory listing, a correctly formed **POST** to `/recurso` with the token in the request body would have returned the password.

**PoC Example**

```bash
curl -s -X POST http://172.17.0.2/recurso   -H 'Content-Type: application/json'   -d '{"token":"tokentraviesito"}'
```

Evidence of the endpoint response:

![Respuesta1](https://i.imgur.com/XmITKDK.png)
![Respuesta2](https://i.imgur.com/A9pU7yY.png)

## Exploitation

A brute-force attack with Hydra was considered to find a username matching the discovered password. After several attempts, the attack succeeded and a valid credential pair was found.

```bash
hydra -L /usr/share/wordlists/rockyou.txt -p "lapassworddebackupmaschingonadetodas" ssh://172.17.0.2:5000 -t 4
```

![Hydra](https://i.imgur.com/UPEkHak.png)

Note: different wordlists were used (Seclists, rockyou, etc.). The idea is to try several lists as long as the target lacks brute-force protection or blacklists.

With username `lovely` and the discovered password, SSH access to the target was obtained:

![Gobuster](https://i.imgur.com/Joocf83.png)

## Privilege Escalation

To escalate privileges, an inventory of binaries with the **SUID** bit set was performed:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/IpIBceK.png)

`/usr/bin/nano` was identified with SUID permissions. In this lab, running nano under SUID allowed editing sensitive system files with elevated privileges.

The following command was used to exploit this condition:

```bash
/usr/bin/nano /etc/passwd -l
```

While editing `/etc/passwd`, the password marker (x) for the root user was removed — a procedure performed only in the lab for demonstration purposes. After saving changes, su was executed and root access was obtained without a password.

![Editar /etc/passwd con nano SUID](https://i.imgur.com/Pz3HDD0.png)
![Obtener root](https://i.imgur.com/QPgVRQU.png)

**Result:** full system compromise (user and root).

## Conclusion

Exposed files, secrets embedded in source code, and a SUID-enabled binary allowed total compromise of the machine. These issues can be avoided through proper deployment controls, secret management, and careful privilege configuration.

---

_Written by **kur0bai**_

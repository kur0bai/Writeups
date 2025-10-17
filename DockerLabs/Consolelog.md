# ConsoleLog

Pequeña Máquina en modo **easy** de Dockerlabs.

- [Reconocimiento](#reconocimiento)
- [Escaneo](#escaneo)
- [Enumeración](#enumeración)
- [Explotación](#explotación)
- [Escalada de privilegios](#escalada-de-privilegios)

<br/>

## Reconocimiento

La máquina objetivo se encuentra correctamente desplegada dentro de la red de laboratorio (en este caso, utilizando Docker).  
Para identificarla se realizó el uso de `arp-scan` para identificar los dispositivos en nuestra red docker con la interfaz `docker0`

```bash
sudo arp-scan -I docker0 --localnet
Interface: docker0, type: EN10MB, MAC: 02:42:77:20:48:b6, IPv4: 172.17.0.1
Starting arp-scan 1.10.0 with 65536 hosts (https://github.com/royhills/arp-scan)
172.17.0.2	02:42:ac:11:00:02	(Unknown: locally administered)
```

<br/>

## Escaneo

Se realizó un escaneo con **Nmap** para identificar puertos abiertos y servicios:

```bash
nmap -p- --open -sC -sV --min-rate 5000 -n -Pn 172.17.0.2
```

El **target** corresponde a la IP de la máquina víctima: **172.17.0.2**

Resultados principales:

- Puerto **5000** abierto para ssh.
- Puerto **80** corriendo una aplicación web con Apache.
- Puerto **3000** corriendo aplicación de nodejs con el framework de express.

![Scan1](https://i.imgur.com/1epCh4F.png)

<br/>

## Enumeración

El siguiente paso fue buscar directorios y recursos ocultos.

Con **Gobuster**:

```bash
gobuster dir -u "http://172.17.0.2"   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -t 20 -x php,txt,html,php.bak
```

El escaneo reveló un directorio que contiene archivos JavaScript del frontend y referencias a un endpoint llamado `/recurso`.

![Gobuster](https://i.imgur.com/zmVJle6.png)

Al inspeccionar la aplicación web en el navegador, se detectó un mensaje en la consola que hacía referencia a un directorio `recurso` y a un token esperado por el backend.

![Console message](https://i.imgur.com/bC2z0KO.png)

Posteriormente, la revisión del servidor mostró que el directorio del backend tenía **Directory Listing** habilitado, permitiendo la lectura directa de ficheros en la raíz del servicio.

![Directory Listing](https://i.imgur.com/QwUb4sa.png)

Dentro de los archivos expuestos se encontró `server.js`, que implementa un endpoint `POST /recurso`.

![server.js](https://i.imgur.com/ZNjSEUB.png)

El endpoint compara un token recibido con un valor **hardcoded** (`tokentraviesito`) y, en caso de coincidencia, devuelve en respuesta una contraseña en texto claro: `lapassworddebackupmaschingonadetodas`.

> **Observación:** incluso sin listar directorios, un `POST` correctamente formado a `/recurso` con el token en el body habría devuelto la contraseña.

Ejemplo PoC (conceptual):

```bash
curl -s -X POST http://172.17.0.2/recurso   -H 'Content-Type: application/json'   -d '{"token":"tokentraviesito"}'
```

Evidencia de la respuesta del endpoint:

![Respuesta1](https://i.imgur.com/XmITKDK.png)
![Respuesta2](https://i.imgur.com/A9pU7yY.png)

<br/>

## Explotación

Se consideraró realizar un ataque de fuerza bruta con Hydra para intentar encontrar el usuario que coincidiera con la password. Afortunadamente, después de varios intentos se tuvo éxito y hubo una coincidencia.

```bash
hydra -L /usr/share/wordlists/rockyou.txt -p "lapassworddebackupmaschingonadetodas" ssh://172.17.0.2:5000 -t 4
```

![Hydra](https://i.imgur.com/UPEkHak.png)

Para tener en cuenta; se utilizaron diferentes diccionarios, desde los clásicos de la seclists hasta el rockyou, la idea es intentar con varios siempre y cuando el target no tenga protección contra bruteforce y blacklists.

Teniendo el usuario `lovely` y su password, se ingresa al ssh para entrar a la máquina.

![Gobuster](https://i.imgur.com/Joocf83.png)

<br/>

## Escalada de privilegios

Para escalar privilegios se realizó un inventario de binarios con el bit **SUID** establecido:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/IpIBceK.png)

Se identificó `/usr/bin/nano` con permisos SUID. En este laboratorio, ejecutar `nano` con SUID permitió editar ficheros sensibles del sistema con privilegios elevados.

Comando empleado para explotar la condición:

```bash
/usr/bin/nano /etc/passwd -l
```

Al editar `/etc/passwd` se eliminó la marca de contraseña (la `x`) del usuario `root` — procedimiento realizado únicamente en el entorno de laboratorio con fines demostrativos. Tras guardar los cambios se ejecutó `su` y se obtuvo acceso como `root` sin requerir contraseña.

![Editar /etc/passwd con nano SUID](https://i.imgur.com/Pz3HDD0.png)
![Obtener root](https://i.imgur.com/QPgVRQU.png)

**Resultado:** compromiso total de la máquina (user + root).

## Conclusión breve

La combinación de ficheros expuestos, secretos embebidos en el código y la presencia de un binario SUID explotable permitieron un compromiso completo de la máquina. Estas fallas son evitables mediante control de despliegue, gestión de secretos y revisión de configuraciones de privilegio.

_Written by **kur0bai**_

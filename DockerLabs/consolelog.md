# Consolelog

### Descripción

Pequeña Máquina en modo **[easy](https://mega.nz/file/oGMWiKoJ#l02GwzicvsgLaczCjTSqaJNl5-NGajklpOY3A3Tu9to)** de Dockerlabs.

---

## 🔎 Reconocimiento

La máquina objetivo se encuentra correctamente desplegada dentro de la red de laboratorio (en este caso, utilizando Docker).  
Dado que la dirección IP es conocida o fácilmente identificable dentro de este entorno controlado, esta fase se clasifica como **reconocimiento pasivo**.

![Reconocimiento](https://i.imgur.com/rs4PSAQ.png)

---

## 📡 Escaneo

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

---

## 📂 Enumeración

El siguiente paso fue buscar directorios y recursos ocultos.

Con **Gobuster**:

```bash
gobuster dir -u "http://172.17.0.2"   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -t 20 -x php,txt,html,php.bak
```

El resultado arrojó información con respecto al backend y un directorio que al parecer contiene los archivos javascript del frontend.

![Gobuster](https://i.imgur.com/zmVJle6.png)

Por otro lado, se inspeccionó la página alojada en el puerto y se encontró una aplicación bastante sencilla pero al inspeccionarla, se hayó un mensaje en consola haciendo referencia a un directorio llamado recurso y un token.

![Gobuster](https://i.imgur.com/bC2z0KO.png)

Al inspeccionar el directorio del servidor backend se encontró una vulnerabilidad de Directory Listing donde se pudo inspeccionar los archivos dentro de la raiz del mismo.

![Gobuster](https://i.imgur.com/QwUb4sa.png)

Revisando el contenido de server.js se encuentra el siguiente código de una API con método POST y el endpoint de `/recurso`

![Gobuster](https://i.imgur.com/ZNjSEUB.png)

Podemos observar que el endpoint no está sanitizado y la clave secreta queda expuesta en respuesta al token esperado: `tokentraviesito` que se encuentra hardcoded, es decir la clave que obtendríamos sería `lapassworddebackupmaschingonadetodas` desde aquí ya podríamos intentar conseguir un usuario que haga match con esta password.

**NOTA:** Si no se hubiera revisado con el Directory Listing, haciendo un post a el endpoint de recurso con el la key de token en el body, igual hubiésemos obtenido la password.

![Gobuster](https://i.imgur.com/XmITKDK.png)
![Gobuster](https://i.imgur.com/A9pU7yY.png)

https://imgur.com/ZNjSEUB

---

## 💥 Explotación

Se consideraró realizar un ataque de fuerza bruta con Hydra para intentar encontrar el usuario que coincidiera con la password. Afortunadamente, después de varios intentos se tuvo éxito y hubo una coincidencia.

![Gobuster](https://i.imgur.com/UPEkHak.png)

Para tener en cuenta; se utilizaron diferentes diccionarios, desde los clásicos de la seclists hasta el rockyou, la idea es intentar con varios siempre y cuando el target no tenga protección contra bruteforce y blacklists.

Teniendo el usuario `lovely` y su password, se ingresa al ssh para entrar a la máquina.

![Gobuster](https://i.imgur.com/Joocf83.png)

---

## 🔝 Escalación de privilegios

Se buscaron binarios con permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/IpIBceK.png)

Aunque algunos binarios no eran explotables, se observó `/usr/bin/nano` en la lista.  
Al ejecutar:

```bash
/usr/bin/nano /etc/passwd -l
```

Se pudo hacer edición del archivo y se removió la "x" del usuario root, esto con el fin de dejarlo sin password.

![SudoNano](https://i.imgur.com/Pz3HDD0.png)

Al guardar nuevamente los cambios, se ejecutó el `su` para conseguir el super user root y al no tener password, el sistema omite el input.

![SudoNano](https://i.imgur.com/QPgVRQU.png)

Con esto la máquina queda completamente pwned.

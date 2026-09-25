---
title: "[ASO] Compilación de un programa en C utilizando un Makefile"
asignatura: ASO
date: 2026-09-25
resumen: "Compilo e instalo GNU Hello 2.12.3 desde el código fuente en una Debian 13: configuro con ./configure --prefix=/opt/hello, compilo con make, paso los tests y reviso qué ficheros quedan instalados. Termino con una desinstalación limpia y compruebo que apt y dpkg nunca llegan a controlar el programa."
tags:
  - Compilación desde fuentes
  - Makefile
  - configure
  - make
  - Debian
---

## Enunciado

> Elige el programa escrito en C que prefieras y comprueba en las fuentes que exista un fichero Makefile o Configure. Deberás compilar desde las fuentes.
> Realiza los pasos necesarios para compilarlo e instálalo en tu equipo en un directorio que no interfiera con tu sistema de paquetes (/opt, /usr/local, etc).
> La corrección se hará en clase y deberás ser capaz de explicar qué son todos los ficheros que se hayan instalado y realizar una desinstalación limpia.

## Programa elegido: GNU Hello

He elegido **GNU Hello** (versión 2.12.3), un programa en C del proyecto GNU pensado precisamente como ejemplo de cómo se compila e instala software desde las fuentes. Aunque es pequeño, incluye el sistema de compilación completo de GNU (`configure` + `Makefile`) e instala varios tipos de ficheros: el ejecutable, su página de manual, un manual en formato info y las traducciones a distintos idiomas.

La práctica la realizo en una máquina virtual **Debian 13 (Trixie)**.

---

## 1. Instalación de las herramientas de compilación

```bash
gabriel@debian-ansible:~$ sudo apt update
gabriel@debian-ansible:~$ sudo apt install build-essential wget
```

- **build-essential**: metapaquete que instala el compilador de C (`gcc`), `make` y las librerías y cabeceras básicas de C.
- **wget**: para descargar el código fuente desde internet.

## 2. Descarga del código fuente

Creo un directorio de trabajo para compilar ahí el programa y descargo el código fuente desde el servidor FTP de GNU:

```bash
gabriel@debian-ansible:~$ mkdir -p ~/compilacion
gabriel@debian-ansible:~$ cd ~/compilacion
gabriel@debian-ansible:~/compilacion$ wget https://ftp.gnu.org/gnu/hello/hello-2.12.3.tar.gz
gabriel@debian-ansible:~/compilacion$ tar -zxvf hello-2.12.3.tar.gz
gabriel@debian-ansible:~/compilacion$ ls -l
total 1168
drwxr-xr-x 10 gabriel gabriel    4096 mar 18  2026 hello-2.12.3
-rw-rw-r--  1 gabriel gabriel 1189208 mar 18  2026 hello-2.12.3.tar.gz
gabriel@debian-ansible:~/compilacion$ cd hello-2.12.3/
```

Opciones de `tar`:

| Opción | Significado |
|---|---|
| `-z` | El fichero está comprimido con gzip |
| `-x` | Extraer |
| `-v` | Mostrar los ficheros según se extraen |
| `-f` | Fichero sobre el que trabajar |

## 3. Comprobación de que existen `configure` y `Makefile`

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ ls
```

Los ficheros más importantes de las fuentes son:

| Fichero | Qué es |
|---|---|
| `configure` | Script que comprueba el sistema y **genera el `Makefile`** |
| `Makefile.in` | Plantilla a partir de la cual `configure` crea el `Makefile` |
| `Makefile.am` | Plantilla de más alto nivel que usa el desarrollador con `automake` |
| `README`, `INSTALL` | Documentación sobre el programa y cómo instalarlo |
| `src/` | Código fuente en C |
| `po/` | Traducciones a otros idiomas |
| `doc/` | Manual info y página man |

Al descargar las fuentes **todavía no existe el `Makefile`**, solo su plantilla `Makefile.in`. El `Makefile` se genera al ejecutar `configure`.

## 4. Configuración de las fuentes

Antes de compilar, ejecutamos el script `configure` incluido en el código fuente:

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ ./configure --prefix=/opt/hello
```

El script `configure` comprueba que el sistema tiene todo lo necesario para compilar el programa (compilador de C, librerías, cabeceras...). Si todo está correcto, va mostrando un listado de comprobaciones (`checking ...`) y al final genera el `Makefile` adaptado a nuestro equipo.

### ¿Por qué `--prefix=/opt/hello`?

La opción `--prefix` indica el directorio donde se instalará el programa al hacer `make install`. Si no se indica, se usa `/usr/local` por defecto.

He elegido `/opt/hello` por dos motivos:

- **No interfiere con el sistema de paquetes**: `apt` y `dpkg` gestionan los ficheros de `/usr`, y en `/opt` no tocan nada.
- **Todos los ficheros quedan en un único directorio**: si lo hubiera instalado en `/usr/local`, los ficheros quedarían repartidos y mezclados con otros programas. Con un directorio propio es muy fácil ver qué se ha instalado y hacer una desinstalación limpia.

### Ficheros que genera `configure`

| Fichero | Qué es |
|---|---|
| `Makefile` | Instrucciones de compilación adaptadas a nuestro sistema |
| `config.h` | Cabecera de C con las características detectadas del sistema |
| `config.log` | Registro de todas las comprobaciones realizadas (útil si algo falla) |
| `config.status` | Script que permite regenerar la configuración |

Podemos comprobar que el prefijo se ha aplicado mirando el `Makefile` generado:

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ grep '^prefix =' Makefile
prefix = /opt/hello
```

## 5. Compilación

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ make
```

`make` lee el `Makefile` y compila el programa: convierte cada fichero de código fuente (`.c`) en un fichero objeto (`.o`) y después los enlaza para crear el ejecutable `src/hello`. En este punto el programa está **compilado pero no instalado**: solo existe dentro de la carpeta de las fuentes.

## 6. Ejecución de los tests

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ make check
...
PASS: tests/atexit-1
PASS: tests/greeting-1
PASS: tests/greeting-2
PASS: tests/hello-1
PASS: tests/last-1
PASS: tests/operand-1
PASS: tests/traditional-1
============================================================================
Testsuite summary for GNU Hello 2.12.3
============================================================================
# TOTAL: 7
# PASS:  7
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
```

`make check` ejecuta las pruebas que incluye el propio programa para comprobar que el binario compilado funciona correctamente. Los 7 tests han pasado sin errores.

## 7. Instalación

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ sudo make install
```

`make install` copia el ejecutable, la página de manual, el manual info y las traducciones al directorio indicado en `--prefix`. Hace falta `sudo` porque `/opt` pertenece a root.

### Resumen de las fases

| Comando | Fase | Qué hace |
|---|---|---|
| `./configure --prefix=/opt/hello` | Configuración | Comprueba el sistema y genera el `Makefile` |
| `make` | Compilación | Genera el ejecutable a partir del código C |
| `make check` | Pruebas | Comprueba que el programa compilado funciona |
| `sudo make install` | Instalación | Copia los ficheros a `/opt/hello` |

---

## 8. Ficheros instalados

Como todo se ha instalado dentro de `/opt/hello`, podemos listar todos los ficheros con un simple `find`:

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ find /opt/hello -type f
/opt/hello/share/locale/ko/LC_MESSAGES/hello.mo
/opt/hello/share/locale/it/LC_MESSAGES/hello.mo
/opt/hello/share/locale/el/LC_MESSAGES/hello.mo
...
/opt/hello/share/locale/es/LC_MESSAGES/hello.mo
...
/opt/hello/share/locale/de/LC_MESSAGES/hello.mo
/opt/hello/share/locale/zh_CN/LC_MESSAGES/hello.mo
/opt/hello/share/locale/da/LC_MESSAGES/hello.mo
/opt/hello/share/man/man1/hello.1
/opt/hello/share/info/hello.info
/opt/hello/bin/hello
```

### Qué es cada fichero

| Fichero | Qué es |
|---|---|
| `/opt/hello/bin/hello` | **El ejecutable**: el programa compilado a partir del código C |
| `/opt/hello/share/man/man1/hello.1` | **Página de manual** que se consulta con `man`. La sección 1 es la de comandos de usuario |
| `/opt/hello/share/info/hello.info` | **Manual en formato info** de GNU, más completo que la página man |
| `/opt/hello/share/locale/XX/LC_MESSAGES/hello.mo` | **Traducciones compiladas**, una por idioma (`es` = español, `fr` = francés, `de` = alemán...). Gracias a ellas el programa muestra los mensajes en el idioma del sistema |

Estructura de directorios:

```
/opt/hello/
├── bin/
│   └── hello                  ← ejecutable
└── share/
    ├── info/
    │   └── hello.info         ← manual info
    ├── locale/
    │   ├── es/LC_MESSAGES/hello.mo
    │   ├── fr/LC_MESSAGES/hello.mo
    │   └── ...                ← una carpeta por idioma
    └── man/
        └── man1/
            └── hello.1        ← página de manual
```

### Dependencias del ejecutable

```bash
gabriel@debian-ansible:~$ ldd /opt/hello/bin/hello
```

El programa solo depende de `libc.so.6`, la librería estándar de C, y del cargador dinámico `ld-linux`.

---

## 9. Comprobación del funcionamiento

### Ejecución del binario

Ejecutamos el programa con su ruta completa, ya que `/opt/hello/bin` no está en el `PATH`:

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ /opt/hello/bin/hello
¡Hola mundo!
```

Lo probamos también con distintos parámetros:

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ /opt/hello/bin/hello --version
hello (GNU Hello) 2.12.3
Copyright (C) 2026 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <https://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Written by Karl Berry, Sami Kerola, Jim Meyering,
and Reuben Thomas.
```

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ /opt/hello/bin/hello --help
Modo de empleo: /opt/hello/bin/hello [OPCIÓN]...
Muestra un saludo amistoso y configurable.

  -t, --traditional       use traditional greeting
  -g, --greeting=TEXT     use TEXT as the greeting message

      --help     display this help and exit
      --version  output version information and exit
...
```

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ /opt/hello/bin/hello -g "Hola Gabriel"
Hola Gabriel
```

La versión que muestra es la **2.12.3**, la que hemos compilado nosotros.

### Uso a través del PATH

Para poder ejecutarlo solo con su nombre, añadimos su carpeta al `PATH` de la sesión actual:

```bash
gabriel@debian-ansible:~$ export PATH=/opt/hello/bin:$PATH
gabriel@debian-ansible:~$ which hello
/opt/hello/bin/hello
gabriel@debian-ansible:~$ hello
¡Hola mundo!
```

Este cambio solo dura mientras la terminal esté abierta, así que no deja nada que haya que deshacer después.

### Página de manual e info

```bash
gabriel@debian-ansible:~$ man -M /opt/hello/share/man hello
gabriel@debian-ansible:~$ info -f /opt/hello/share/info/hello.info
```

La opción `-M` le indica a `man` en qué directorio buscar, porque por defecto no busca en `/opt/hello`.

### Traducciones

```bash
gabriel@debian-ansible:~$ LANGUAGE=fr hello
gabriel@debian-ansible:~$ LANGUAGE=de hello
gabriel@debian-ansible:~$ LANGUAGE=it hello
```

Cada ejecución muestra el saludo en un idioma diferente, usando los ficheros `.mo` de `share/locale`.

### Comprobación de que no interfiere con el sistema de paquetes

Con `apt policy` vemos que, para el sistema de paquetes, `hello` **no está instalado**. Además, la versión de los repositorios de Debian es la 2.10, mientras que la nuestra es la 2.12.3:

```bash
gabriel@debian-ansible:~$ apt policy hello
hello:
  Instalados: (ninguno)
  Candidato:  2.10-5
  Tabla de versión:
     2.10-5 500
        500 http://deb.debian.org/debian trixie/main amd64 Packages
```

Con `dpkg -S` buscamos qué paquete ha instalado un fichero concreto. `dpkg` guarda en su base de datos todos los ficheros que instala cada paquete. Como nuestro binario no se ha instalado con `apt`, no lo encuentra; en cambio, con un fichero del sistema como `ls`, sí nos dice su paquete:

```bash
gabriel@debian-ansible:~$ dpkg -S /opt/hello/bin/hello
dpkg-query: no se ha encontrado ningún paquete que corresponda con el patrón /opt/hello/bin/hello.

gabriel@debian-ansible:~$ dpkg -S /usr/bin/ls
coreutils: /usr/bin/ls
```

Esto demuestra que el programa está **fuera del sistema de paquetes**: `apt` y `dpkg` no lo controlan ni lo van a modificar al actualizar el sistema. Por el mismo motivo, la desinstalación hay que hacerla a mano.

---

## 10. Desinstalación limpia

### Paso 1: `make uninstall`

El `Makefile` incluye un objetivo `uninstall` que borra los ficheros que instaló `make install`. Hay que ejecutarlo desde la carpeta de las fuentes, ya configurada con el mismo `--prefix`:

```bash
gabriel@debian-ansible:~$ cd ~/compilacion/hello-2.12.3
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ sudo make uninstall
```

### Paso 2: comprobar qué ha quedado

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ find /opt/hello
```

`make uninstall` borra los ficheros, pero **no borra los directorios** que creó la instalación, así que quedan carpetas vacías. Podemos comprobar que ya no queda ningún fichero:

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ find /opt/hello -type f
```

### Paso 3: borrar los directorios vacíos

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ sudo rm -rf /opt/hello
```

Como hemos usado un prefijo propio, podemos borrar el directorio entero sin riesgo de eliminar ficheros de otros programas.

### Paso 4: comprobar que no queda nada

```bash
gabriel@debian-ansible:~$ ls /opt
gabriel@debian-ansible:~$ hash -r
gabriel@debian-ansible:~$ which hello || printf "hello ya no está instalado\n"
hello ya no está instalado
```

`hash -r` hace que bash olvide la ruta del programa que tenía guardada en caché.

### Paso 5: limpiar las fuentes (opcional)

```bash
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ make distclean
gabriel@debian-ansible:~/compilacion/hello-2.12.3$ cd ~
gabriel@debian-ansible:~$ rm -rf ~/compilacion
```

| Comando | Qué borra |
|---|---|
| `make clean` | Solo lo generado al compilar (`.o` y ejecutable) |
| `make distclean` | Además, lo generado por `configure` (`Makefile`, `config.h`, `config.log`, `config.status`) |

---

## Conclusión

El proceso de compilación desde las fuentes con las herramientas de GNU tiene tres fases:

1. **Configurar** (`./configure`): comprueba el sistema y genera el `Makefile`. Con `--prefix` elegimos dónde instalar.
2. **Compilar** (`make`): genera el ejecutable a partir del código C.
3. **Instalar** (`make install`): copia los ficheros a su destino.

Instalar en un directorio propio como `/opt/hello` evita conflictos con el sistema de paquetes, ya que `apt` y `dpkg` gestionan `/usr`, y facilita tanto revisar los ficheros instalados como hacer una desinstalación completamente limpia.

---
title: "Configuración Apache2 PHP-FPM"
asignatura: ImpAppWeb
date: 2026-10-15
resumen: "Cambio la forma en que Apache ejecuta PHP: desinstalo mod_php, instalo PHP-FPM y configuro Apache como proxy inverso con proxy_fcgi, primero por socket UNIX y después por 127.0.0.1:9000, compruebo que la aplicación Biblioteca sigue funcionando y subo memory_limit a 256M en el php.ini de PHP-FPM."
tags:
  - PHP-FPM
  - Apache
  - PHP
  - FastCGI
  - Debian
---

|                      |                                                                                   |
| -------------------- | --------------------------------------------------------------------------------- |
| **Alumno**           | Gabriel Merencio Ortega                                                           |
| **Curso**            | 2º ASIR — I.E.S. Gonzalo Nazareno                                                 |
| **Módulo**           | Implantación de Aplicaciones Web                                                  |
| **Servidor**         | Debian 13 (Trixie) — Apache 2.4 + PHP 8.4 + MariaDB                               |
| **Punto de partida** | Aplicación Biblioteca funcionando en `biblioteca.gabriel.org` (práctica anterior) |

## Objetivo

Cambiar la arquitectura con la que Apache ejecuta PHP:

| Arquitectura                    | Cómo funciona                                                                                                                                   |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Apache2 + mod_php** (antes)   | Cada proceso de Apache lleva incrustado un intérprete PHP y ejecuta él mismo los `.php`. Es sencillo, pero consume más memoria.                 |
| **Apache2 + PHP-FPM** (después) | Apache sirve el contenido estático y actúa como **proxy inverso**: reenvía los `.php` a PHP-FPM por FastCGI y devuelve la respuesta al cliente. |

**PHP-FPM** (*FastCGI Process Manager*) es un servidor de aplicaciones PHP que funciona como servicio independiente, con su propio *pool* de procesos. Primero se conectará con Apache por un socket UNIX y después por el socket TCP `127.0.0.1:9000`.

---

## 1. Desinstalar mod_php

El módulo que permite a Apache ejecutar PHP es `mod_php` (paquete `libapache2-mod-php8.4`).

### 1.1. Situación inicial

```bash
apache2ctl -M | grep php
dpkg -l | grep libapache2-mod-php
```

![mod_php cargado e instalado](/assets/img/ImpAppWeb/2026-10-15-12-18-35-image.png)

### 1.2. Deshabilitar y desinstalar

```bash
sudo a2dismod php8.4
sudo apt purge libapache2-mod-php8.4 libapache2-mod-php -y
sudo systemctl restart apache2
```

`a2dismod` deshabilita el módulo en Apache y `apt purge` lo desinstala junto con su configuración. También se quitan los metapaquetes `php` y `php8.4`, pero el intérprete y las extensiones se mantienen.

### 1.3. Comprobación

```bash
apache2ctl -M | grep php
```

No devuelve nada: Apache ya no ejecuta PHP.

![mod_php desinstalado](/assets/img/ImpAppWeb/2026-10-15-12-19-38-image.png)

---

## 2. Instalación de PHP-FPM

### 2.1. Instalación

```bash
sudo apt install php-fpm php
dpkg -l | grep -E "php8.4-fpm|libapache2-mod-php"
```

Se comprueba que se ha instalado `php8.4-fpm` y que `libapache2-mod-php8.4` no ha vuelto.

![php8.4-fpm instalado sin mod_php](/assets/img/ImpAppWeb/2026-10-15-12-22-18-image.png)

### 2.2. Ficheros de configuración

| Ruta                               | Para qué sirve                     |
| ---------------------------------- | ---------------------------------- |
| `/etc/php/8.4/fpm/conf.d/`         | Módulos de PHP activos en PHP-FPM  |
| `/etc/php/8.4/fpm/php-fpm.conf`    | Configuración general del servicio |
| `/etc/php/8.4/fpm/php.ini`         | Configuración de PHP para PHP-FPM  |
| `/etc/php/8.4/fpm/pool.d/www.conf` | Pool de procesos por defecto       |

```bash
grep -E "^\[|^user|^group|^listen =|^pm =" /etc/php/8.4/fpm/pool.d/www.conf
```

El pool `[www]` ejecuta los procesos como `www-data`, escucha en el socket `/run/php/php8.4-fpm.sock` y gestiona los procesos de forma dinámica (`pm = dynamic`).

![Parámetros del pool www](/assets/img/ImpAppWeb/2026-10-15-12-23-06-image.png)

### 2.3. Procesos PHP-FPM

```bash
sudo systemctl status php8.4-fpm
ps aux | grep php-fpm
ls -l /run/php/
```

Hay un proceso **master** de `root`, que gestiona el pool, y varios procesos **pool www** de `www-data`, que ejecutan el código PHP. En `/run/php/` está el socket `php8.4-fpm.sock`.

> **Entregable 1** — Procesos PHP-FPM corriendo en el servidor.

![Procesos PHP-FPM corriendo](/assets/img/ImpAppWeb/2026-10-15-12-23-44-image.png)

---

## 3. Configurar Apache2 para usar PHP-FPM

Se configura **para todos los virtual host**, usando el fichero que trae el paquete `php8.4-fpm`.

### 3.1. Módulos necesarios

```bash
sudo a2enmod proxy_fcgi setenvif
```

| Módulo       | Para qué sirve                                                            |
| ------------ | ------------------------------------------------------------------------- |
| `proxy_fcgi` | Permite a Apache hacer de proxy hacia PHP-FPM usando FastCGI              |
| `setenvif`   | Lo usa la configuración de PHP-FPM para pasar la cabecera `Authorization` |

### 3.2. Activar la configuración de PHP-FPM

```bash
sudo a2enconf php8.4-fpm
sudo apache2ctl configtest
sudo systemctl restart apache2
```

> **Entregable 2** — Configuración de Apache2 para trabajar con PHP-FPM.

![Activación de php8.4-fpm en Apache](/assets/img/ImpAppWeb/2026-10-15-12-28-03-image.png)

Contenido de `/etc/apache2/conf-available/php8.4-fpm.conf` sin comentarios:

```bash
grep -v "^\s*#" /etc/apache2/conf-available/php8.4-fpm.conf | grep -v "^\s*$"
```

![Contenido de php8.4-fpm.conf](/assets/img/ImpAppWeb/2026-10-15-12-31-21-image.png)

| Parte                                                                | Qué hace                                           |
| -------------------------------------------------------------------- | -------------------------------------------------- |
| `<IfModule !mod_php8.c>`                                             | Solo se aplica si `mod_php` no está cargado        |
| `<FilesMatch ".+\.ph(?:ar\|p\|tml)$">`                               | Selecciona los ficheros `.php`, `.phtml` y `.phar` |
| `SetHandler "proxy:unix:/run/php/php8.4-fpm.sock\|fcgi://localhost"` | Envía esos ficheros a PHP-FPM por el socket UNIX   |
| `Require all denied`                                                 | Impide descargar el código fuente PHP (`.phps`)    |

### 3.3. Comprobación

```bash
apache2ctl -M | grep -E "proxy|setenvif"
ls -l /etc/apache2/conf-enabled/ | grep fpm
```

![Módulos cargados y configuración habilitada](/assets/img/ImpAppWeb/2026-10-15-12-32-19-image.png)

### 3.4. Alternativa: configuración en cada virtual host

También se podría añadir en cada `<VirtualHost>` uno de estos bloques, lo que permite usar un pool distinto por sitio:

```apache
<FilesMatch "\.php$">
    SetHandler "proxy:unix:/run/php/php8.4-fpm.sock|fcgi://127.0.0.1/"
</FilesMatch>
```

```apache
ProxyPassMatch ^/(.*\.php)$ unix:/run/php/php8.4-fpm.sock|fcgi://127.0.0.1/var/www/biblioteca
```

---

## 4. Comprobación con socket UNIX

### 4.1. PHP se ejecuta con PHP-FPM

```bash
printf "<?php\nphpinfo();\n?>\n" | sudo tee /var/www/biblioteca/info.php
```

En `http://biblioteca.gabriel.org/info.php`:

| Parámetro                     | Valor                      | Significado                                                     |
| ----------------------------- | -------------------------- | --------------------------------------------------------------- |
| **Server API**                | `FPM/FastCGI`              | PHP lo ejecuta PHP-FPM (con mod_php ponía `Apache 2.0 Handler`) |
| **Loaded Configuration File** | `/etc/php/8.4/fpm/php.ini` | Se aplica el `php.ini` de PHP-FPM                               |

> **Entregable 3** — `info.php` mostrando que PHP se ejecuta con PHP-FPM.

![Server API FPM/FastCGI](/assets/img/ImpAppWeb/2026-10-15-12-37-17-image.png)

### 4.2. La aplicación Biblioteca sigue funcionando

Inicio de sesión con `admin` / `admin` y acceso a **Libros**. Los `.php` los ejecuta PHP-FPM; el CSS, el JS y las imágenes los sigue sirviendo Apache.

> **Entregable 4** — Biblioteca funcionando después del login (socket UNIX).

![Panel de administración](/assets/img/ImpAppWeb/2026-10-15-12-38-03-image.png)

![Listado de libros](/assets/img/ImpAppWeb/2026-10-15-12-38-28-image.png)

---

## 5. PHP-FPM en el puerto tcp/9000

Se cambia la comunicación del socket UNIX a un socket TCP. Hay que modificar los dos extremos: dónde escucha PHP-FPM y adónde envía Apache las peticiones.

### 5.1. Configuración de PHP-FPM

En `/etc/php/8.4/fpm/pool.d/www.conf`:

```ini
listen = 127.0.0.1:9000
```

Se usa `127.0.0.1:9000` en lugar de `listen = 9000` para que PHP-FPM solo acepte conexiones del propio servidor y no de toda la red.

> **Entregable 5** — Configuración de PHP-FPM.

![Directiva listen en www.conf](/assets/img/ImpAppWeb/2026-10-15-12-41-08-image.png)

```bash
sudo systemctl restart php8.4-fpm
grep "^listen =" /etc/php/8.4/fpm/pool.d/www.conf
sudo ss -tlnp | grep 9000
```

PHP-FPM escucha en `127.0.0.1:9000` (estado `LISTEN`, proceso `php-fpm8.4`).

![PHP-FPM escuchando en el puerto 9000](/assets/img/ImpAppWeb/2026-10-15-12-42-08-image.png)

### 5.2. Configuración de Apache2

En `/etc/apache2/conf-available/php8.4-fpm.conf`, dentro del bloque `<FilesMatch>`:

```apache
# Antes (socket UNIX)
SetHandler "proxy:unix:/run/php/php8.4-fpm.sock|fcgi://localhost"

# Después (socket TCP)
SetHandler "proxy:fcgi://127.0.0.1:9000"
```

> **Entregable 5** — Configuración de Apache2.

![SetHandler apuntando al puerto 9000](/assets/img/ImpAppWeb/2026-10-15-12-43-42-image.png)

```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
grep "SetHandler" /etc/apache2/conf-available/php8.4-fpm.conf
```

![Comprobación del SetHandler](/assets/img/ImpAppWeb/2026-10-15-12-44-34-image.png)

---

## 6. Comprobación con el puerto tcp/9000

### 6.1. La comunicación va por TCP

```bash
ls -l /run/php/
```

El socket `php8.4-fpm.sock` ha desaparecido. El `php-fpm.sock` que queda es un enlace simbólico del sistema de alternativas de Debian que apuntaba a él, y ahora está roto (aparece en rojo).

![Socket UNIX eliminado](/assets/img/ImpAppWeb/2026-10-15-12-47-49-image.png)

Tras recargar una página de la aplicación:

```bash
sudo ss -tan | grep :9000
```

Además de la línea `LISTEN`, aparecen conexiones `TIME-WAIT` con `127.0.0.1:9000`: son las que abre Apache para pasar cada petición PHP a PHP-FPM. Se usa `-a` porque sin ella `ss` no muestra ese estado.

![Conexiones de Apache a PHP-FPM por el puerto 9000](/assets/img/ImpAppWeb/2026-10-15-12-50-52-image.png)

### 6.2. La aplicación sigue funcionando

El **Server API** sigue siendo `FPM/FastCGI`:

![info.php con PHP-FPM por TCP](/assets/img/ImpAppWeb/2026-10-15-12-51-21-image.png)

> — Biblioteca funcionando después del login (puerto tcp/9000).

![Panel de administración por TCP](/assets/img/ImpAppWeb/2026-10-15-12-52-11-image.png)

![Listado de libros por TCP](/assets/img/ImpAppWeb/2026-10-15-12-52-22-image.png)

---

## 7. Cambiar memory_limit a 256M

### 7.1. Fichero a modificar

> **Entregable 6** — El fichero modificado es **`/etc/php/8.4/fpm/php.ini`**.

Es el `php.ini` que usa PHP-FPM, como indica *Loaded Configuration File* en `info.php`. El de `apache2` era el de mod_php (ya eliminado) y el de `cli` solo afecta a la terminal.

```bash
grep "^memory_limit" /etc/php/8.4/fpm/php.ini
```

![memory_limit antes del cambio](/assets/img/ImpAppWeb/2026-10-15-12-54-21-image.png)

### 7.2. Modificación

```ini
memory_limit = 256M
```

Como el `php.ini` lo lee PHP-FPM y no Apache, se reinicia `php8.4-fpm`:

```bash
sudo systemctl restart php8.4-fpm
grep "^memory_limit" /etc/php/8.4/fpm/php.ini
```

![memory_limit a 256M](/assets/img/ImpAppWeb/2026-10-15-12-55-18-image.png)

### 7.3. Comprobación en info.php

`memory_limit` vale `256M` en *Local Value* y *Master Value*, cargado desde `/etc/php/8.4/fpm/php.ini`.

> **Entregable 6** — `info.php` mostrando el cambio.

![Loaded Configuration File de FPM](/assets/img/ImpAppWeb/2026-10-15-12-56-23-image.png)

![memory_limit 256M en info.php](/assets/img/ImpAppWeb/2026-10-15-12-56-47-image.png)

Al terminar se borra `info.php`, porque muestra información sensible del servidor:

```bash
sudo rm /var/www/biblioteca/info.php
```

---

## Conclusión

Se ha pasado de **Apache2 + mod_php** a **Apache2 + PHP-FPM**, con Apache como proxy inverso hacia PHP-FPM, primero por socket UNIX y después por `127.0.0.1:9000`. La aplicación Biblioteca ha funcionado en ambos casos, y el límite de memoria se ha configurado en el `php.ini` de PHP-FPM.

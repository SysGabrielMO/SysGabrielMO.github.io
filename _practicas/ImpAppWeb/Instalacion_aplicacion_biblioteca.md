---
title: "Instalación aplicación biblioteca"
asignatura: ImpAppWeb
date: 2026-10-09
resumen: "Instalo una aplicación PHP de gestión de biblioteca sobre la pila LAMP: creo su base de datos y un usuario con privilegios solo sobre ella, la publico con un VirtualHost propio, configuro su conexión a MariaDB, activo mod_rewrite con AllowOverride All y subo memory_limit a 256M en el php.ini de Apache."
tags:
  - PHP
  - Apache
  - MariaDB
  - VirtualHost
  - mod_rewrite
  - Debian
---

**Alumno:** Gabriel Merencio Ortega  
**Curso:** 2º ASIR — I.E.S. Gonzalo Nazareno  
**Servidor:** Debian 13 (Trixie) con pila LAMP (Apache + MariaDB + PHP 8.4)  
**Aplicación:** [Sistema de biblioteca básico PHP 8 y MySQL](https://github.com/VidaInformatico/Sistema-de-biblioteca-basico-php-8-y-mysql)  

En esta práctica se instala una aplicación web de gestión de biblioteca sobre el servidor LAMP montado en la práctica anterior: se crea su base de datos, se publica con un virtual host propio, se configura su conexión a la base de datos, se activa el módulo `rewrite` de Apache, se accede desde el cliente y se ajusta la configuración de PHP.

---

## 1. Crear la base de datos

### 1.1. Descargar la aplicación

Descargamos el código de la aplicación en el directorio personal del usuario. En él viene el fichero `biblioteca.sql` con el esquema de las tablas.

```bash
sudo apt install git -y
cd ~
git clone https://github.com/VidaInformatico/Sistema-de-biblioteca-basico-php-8-y-mysql.git
ls Sistema-de-biblioteca-basico-php-8-y-mysql
```



### 1.2. Crear la base de datos y el usuario

Entramos en MariaDB como `root`:

```bash
sudo mariadb -u root -p
```

Creamos la base de datos `biblioteca` y un usuario `biblioteca` que solo tiene privilegios sobre ella:

```sql
CREATE DATABASE biblioteca;
CREATE USER 'biblioteca'@'localhost' IDENTIFIED BY 'Biblio_2026';
GRANT ALL PRIVILEGES ON biblioteca.* TO 'biblioteca'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

| Sentencia                              | Qué hace                                                                        |
| -------------------------------------- | ------------------------------------------------------------------------------- |
| `CREATE DATABASE`                      | Crea la base de datos vacía de la aplicación                                    |
| `CREATE USER`                          | Crea el usuario que usará la aplicación para conectarse, solo desde `localhost` |
| `GRANT ALL PRIVILEGES ON biblioteca.*` | Le da todos los permisos, pero únicamente sobre esa base de datos               |
| `FLUSH PRIVILEGES`                     | Recarga la tabla de privilegios para que los cambios se apliquen                |

![](/assets/img/ImpAppWeb/2026-10-09-12-35-10-image.png)



### 1.3. Importar las tablas

El fichero `biblioteca.sql` solo contiene las tablas y sus datos; no crea la base de datos ni la selecciona con `USE`. Por eso se indica la base de datos destino en el propio comando:

```bash
mariadb -u biblioteca -p biblioteca < ~/Sistema-de-biblioteca-basico-php-8-y-mysql/biblioteca.sql
```

El primer `biblioteca` es el usuario y el segundo, la base de datos donde se importan las tablas. Se hace con el usuario nuevo para comprobar a la vez que sus privilegios funcionan.

### 1.4. Comprobación

```bash
mariadb -u biblioteca -p biblioteca -e "SHOW TABLES;"
```

Aparecen las 10 tablas de la aplicación: `autor`, `configuracion`, `detalle_permisos`, `editorial`, `estudiante`, `libro`, `materia`, `permisos`, `prestamo` y `usuarios`.



![](/assets/img/ImpAppWeb/2026-10-09-12-36-15-image.png)



---

## 2. Crear el virtual host

Se accederá con el nombre `biblioteca.gabriel.org` y los ficheros de la aplicación se ubicarán en el directorio `/var/www/biblioteca`, que será el DocumentRoot del sitio.

### 2.1. Copiar los ficheros de la aplicación

Creamos el directorio y copiamos dentro el contenido del repositorio. Se usa `/.` al final del origen para que se copien también los ficheros ocultos, entre ellos el `.htaccess`, que la aplicación necesita:

```bash
sudo mkdir -p /var/www/biblioteca
sudo cp -r ~/Sistema-de-biblioteca-basico-php-8-y-mysql/. /var/www/biblioteca/
```

Borramos la carpeta `.git`, que no hace falta en el servidor y expondría el historial del repositorio a través de la web:

```bash
sudo rm -rf /var/www/biblioteca/.git
```

Damos la propiedad al usuario de Apache, `www-data`, porque la aplicación necesita escribir en su carpeta (por ejemplo, al subir las imágenes de libros y autores):

```bash
sudo chown -R www-data:www-data /var/www/biblioteca
ls -la /var/www/biblioteca
```

![](/assets/img/ImpAppWeb/2026-10-09-12-38-55-image.png)



### 2.2. Crear el virtual host

Creamos el fichero de configuración del sitio:

```bash
cd /etc/apache2/sites-available
sudo nano /etc/apache2/sites-available/biblioteca.conf
```

```apache
<VirtualHost *:80>
    ServerName biblioteca.gabriel.org
    DocumentRoot /var/www/biblioteca

    ErrorLog ${APACHE_LOG_DIR}/biblioteca_error.log
    CustomLog ${APACHE_LOG_DIR}/access_biblioteca.log combined
</VirtualHost>
```

| Directiva                | Qué hace                                                             |
| ------------------------ | -------------------------------------------------------------------- |
| `ServerName`             | Nombre por el que Apache reconoce que la petición es para este sitio |
| `DocumentRoot`           | Directorio donde están los ficheros de la web                        |
| `ErrorLog` / `CustomLog` | Logs de errores y de acceso propios de este sitio                    |

### 2.3. Habilitar el sitio

```bash
sudo a2ensite biblioteca.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

`a2ensite` crea el enlace del sitio en `sites-enabled` y `configtest` comprueba la sintaxis antes de recargar Apache; debe responder `Syntax OK`.

![](/assets/img/ImpAppWeb/2026-10-09-12-41-56-image.png)



---

## 3. Configurar el acceso a la base de datos desde la aplicación

La aplicación lee los datos de conexión y su propia URL del fichero `Config/Config.php`. Por defecto viene preparada para un XAMPP local, con el usuario `root` sin contraseña y la URL `http://localhost/biblio/`:

```php
<?php
const base_url = "http://localhost/biblio/";
const host = "localhost";
const user = "root";
const pass = "";
const db = "biblioteca";
const charset = "charset=utf8";
?>
```

Lo editamos con los datos de nuestro servidor:

```bash
sudo nano /var/www/biblioteca/Config/Config.php
```

```php
<?php
const base_url = "http://biblioteca.gabriel.org/";
const host = "localhost";
const user = "biblioteca";
const pass = "Biblio_2026";
const db = "biblioteca";
const charset = "charset=utf8";
?>
```

| Constante  | Valor                                          | Qué es                                                                                                  |
| ---------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `base_url` | `http://biblioteca.gabriel.org/` | URL con la que se accede a la aplicación; la usa para construir los enlaces y cargar CSS, JS e imágenes |
| `host`     | `localhost`                                    | Servidor donde está la base de datos (la misma máquina)                                                 |
| `user`     | `biblioteca`                                   | Usuario creado en el punto 1                                                                            |
| `pass`     | `Biblio_2026`                                  | Contraseña de ese usuario                                                                               |
| `db`       | `biblioteca`                                   | Base de datos creada en el punto 1                                                                      |

La `base_url` debe terminar en `/`, porque la aplicación le concatena directamente las rutas (`Assets/...`, `Usuarios/...`). Sin la barra, los enlaces quedarían mal formados.


### Comprobación

Comprobamos que el fichero no tiene errores de sintaxis PHP:

```bash
php -l /var/www/biblioteca/Config/Config.php
```

Debe responder `No syntax errors detected`.



![](/assets/img/ImpAppWeb/2026-10-09-12-45-11-image.png)

![](/assets/img/ImpAppWeb/2026-10-09-12-45-27-image.png)



---

## 4. Activación del módulo rewrite

La aplicación usa URLs "amigables" como `/Libros` o `/Usuarios/listar`, que no corresponden a ficheros reales. El módulo `rewrite` de Apache permite que, al pedir una de esas URLs, el servidor ejecute internamente otra. La regla está en el fichero `.htaccess` de la aplicación:

```apache
RewriteEngine on
RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^(.*)$ index.php?url=$1 [QSA,L]
```

| Línea                 | Qué hace                                                                                                                                   |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `RewriteEngine on`    | Activa la reescritura de URLs en este directorio                                                                                           |
| `RewriteCond ... !-d` | Solo se aplica si lo pedido no es un directorio que exista                                                                                 |
| `RewriteCond ... !-f` | Solo se aplica si lo pedido no es un fichero que exista (así CSS, JS e imágenes se sirven normalmente)                                     |
| `RewriteRule`         | Todo lo demás se envía a `index.php`, pasando la ruta en el parámetro `url`. Por ejemplo, `/Libros` se convierte en `index.php?url=Libros` |

### 4.1. Activar el módulo rewrite

```bash
sudo a2enmod rewrite
sudo systemctl restart apache2
```

Comprobamos que está cargado:

```bash
apache2ctl -M | grep rewrite
```

Debe aparecer `rewrite_module (shared)`.

![](/assets/img/ImpAppWeb/2026-10-09-12-48-11-image.png)

### 4.2. Permitir el uso de .htaccess

Por defecto Debian ignora los ficheros `.htaccess` en `/var/www/`, porque la directiva `AllowOverride` está a `None`. Si no lo cambiamos, la regla anterior no se aplica aunque el módulo esté activo.

```bash
sudo nano /etc/apache2/apache2.conf
```

Buscamos el bloque del directorio `/var/www/` y cambiamos `None` por `All`:



```apache
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>
```

`AllowOverride All` permite que los `.htaccess` de los directorios bajo `/var/www/` modifiquen la configuración de Apache para ese directorio.

Comprobamos la sintaxis y reiniciamos:

```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
```

![](/assets/img/ImpAppWeb/2026-10-09-12-50-45-image.png)



### 4.3. Comprobación

```bash
grep -A 4 "<Directory /var/www/>" /etc/apache2/apache2.conf
sudo systemctl status apache2
```

![](/assets/img/ImpAppWeb/2026-10-09-12-59-52-image.png)



---

## 5. Acceso a la aplicación

El nombre `biblioteca.gabriel.org` no existe en ningún DNS, así que el cliente no sabe a qué IP corresponde. Para resolverlo se añade una entrada en el fichero `hosts` del cliente, que el sistema consulta antes que el DNS.

### 5.1. Resolución de nombres en el cliente

En el cliente (el host anfitrión) editamos el fichero `hosts`:

```bash
sudo nano /etc/hosts
```

Y añadimos una línea con la IP del servidor y el nombre del sitio:

![](/assets/img/ImpAppWeb/2026-10-09-12-54-24-image.png)



### 5.2. Acceso a la aplicación

Desde el navegador del cliente entramos en:

```
http://biblioteca.gabriel.org
```

Aparece la pantalla de inicio de sesión de la aplicación.

![](/assets/img/ImpAppWeb/2026-10-09-12-55-52-image.png)

Iniciamos sesión con el usuario `admin` y la contraseña `admin`, y accedemos al panel de administración.

Para comprobar que la aplicación lee correctamente de la base de datos y que el módulo rewrite funciona, entramos en la sección **Libros**: la URL es `/Libros` y muestra los libros que venían en `biblioteca.sql`.

![](/assets/img/ImpAppWeb/2026-10-09-13-00-48-image.png)



![](/assets/img/ImpAppWeb/2026-10-09-13-01-04-image.png)



### 5.3. Logs del sitio

En el servidor se pueden ver las peticiones que llegan al virtual host:

```bash
sudo tail /var/log/apache2/access_biblioteca.log
```

![](/assets/img/ImpAppWeb/2026-10-09-13-02-19-image.png)

---

## 6. Cambiar la configuración de PHP

El parámetro `memory_limit` indica la memoria RAM máxima que puede usar un script PHP. Por defecto vale `128M`; lo subimos a `256M`.

### 6.1. ¿En qué fichero se cambia?

En Debian, PHP tiene un `php.ini` distinto para cada forma de ejecutarse:

| Fichero                        | Se usa cuando…                                       |
| ------------------------------ | ---------------------------------------------------- |
| `/etc/php/8.4/apache2/php.ini` | PHP se ejecuta dentro de Apache (las páginas web)    |
| `/etc/php/8.4/cli/php.ini`     | PHP se ejecuta desde la terminal (`php fichero.php`) |

Como la aplicación se ejecuta a través de Apache con `libapache2-mod-php`, el fichero que hay que modificar es **`/etc/php/8.4/apache2/php.ini`**. Cambiar el de `cli` no afectaría a la web.

Comprobamos el valor actual:

```bash
grep "^memory_limit" /etc/php/8.4/apache2/php.ini
```

![](/assets/img/ImpAppWeb/2026-10-09-13-05-47-image.png)



### 6.2. Modificar el parámetro

```bash
sudo nano /etc/php/8.4/apache2/php.ini
```

Buscamos la línea `memory_limit` y la dejamos así:

```ini
memory_limit = 256M
```

![](/assets/img/ImpAppWeb/2026-10-09-13-06-30-image.png)

En PHP la unidad se escribe con `M` (megabytes), sin la `b`.

Reiniciamos Apache para que vuelva a leer el `php.ini`:

```bash
sudo systemctl restart apache2
grep "^memory_limit" /etc/php/8.4/apache2/php.ini
```

![](/assets/img/ImpAppWeb/2026-10-09-13-06-59-image.png)



### 6.3. Comprobación en info.php

Creamos un `info.php` en el DocumentRoot de la aplicación:


```bash
printf "<?php\nphpinfo();\n?>\n" | sudo tee /var/www/biblioteca/info.php
```

Accedemos desde el cliente a `http://biblioteca.gabriel.org/info.php`. Como `info.php` es un fichero que existe, la regla del `.htaccess` no lo redirige a `index.php` y se ve directamente.

En la página comprobamos:

- **Loaded Configuration File**: `/etc/php/8.4/apache2/php.ini`, lo que confirma que es el fichero que usa Apache.
- **memory_limit**: `256M`, tanto en *Local Value* como en *Master Value*.



![](/assets/img/ImpAppWeb/2026-10-09-13-08-56-image.png)



![](/assets/img/ImpAppWeb/2026-10-09-13-08-23-image.png)

Al terminar la comprobación se borra el fichero, porque muestra información sensible del servidor a cualquiera que acceda:

```bash
sudo rm /var/www/biblioteca/info.php
```

---

## Conclusión

Se ha instalado la aplicación de biblioteca sobre la pila LAMP: se ha creado su base de datos con un usuario propio con privilegios solo sobre ella, se ha publicado con el virtual host `biblioteca.gabriel.org`, se ha configurado su conexión a MariaDB en `Config/Config.php`, se ha activado `mod_rewrite` permitiendo el uso de `.htaccess` con `AllowOverride All`, se ha accedido desde el cliente mediante el fichero `hosts` y se ha aumentado `memory_limit` a 256M en el `php.ini` de Apache, comprobándolo con `phpinfo()`.

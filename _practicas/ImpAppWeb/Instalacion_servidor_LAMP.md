---
title: "Instalación servidor LAMP"
asignatura: ImpAppWeb
date: 2026-10-08
resumen: "Instalo la pila LAMP en Debian 13 (Apache, MariaDB y PHP 8.4), compruebo que PHP se ejecuta en Apache y conecta con la base de datos, publico el sitio gabrielmerencioortega.com con un VirtualHost accesible desde el host y consulto los logs de acceso, de errores y del servicio."
tags:
  - LAMP
  - Apache
  - MariaDB
  - PHP
  - VirtualHost
  - Debian
---

**Alumno:** Gabriel Merencio Ortega  
**Curso:** 2º ASIR — I.E.S. Gonzalo Nazareno  
**Sistema del servidor:** Debian 13 (Trixie)

En esta práctica se instala la pila **LAMP** (Linux + Apache + MariaDB/MySQL + PHP) en un servidor Debian, se comprueba su funcionamiento, se publica un sitio web accesible desde el host con el nombre `gabrielmerencioortega.com` y se consultan los logs del servicio.

---

## 0. Preparación del sistema

Actualizamos la lista de paquetes y el sistema antes de instalar nada:

```bash
sudo apt update
sudo apt upgrade -y
```

Comprobamos la IP del servidor, que usaremos más adelante desde el host:

```bash
ip -c a
```

![](/assets/img/ImpAppWeb/2026-10-08-13-32-06-image.png)

---

## 1. Instalación de la base de datos (MariaDB)

En Debian el paquete `mysql-server` ha sido sustituido por **MariaDB**, un fork compatible con MySQL.

```bash
sudo apt install mariadb-server mariadb-client -y
```

![](/assets/img/ImpAppWeb/2026-10-08-13-15-39-image.png)

Comprobamos que el servicio está activo y habilitado en el arranque:

```bash
sudo systemctl status mariadb
sudo systemctl is-enabled mariadb
```

### Securizar la instalación

```bash
sudo mariadb-secure-installation
```

Respuestas recomendadas:

| Pregunta                             | Respuesta               |
| ------------------------------------ | ----------------------- |
| Enter current password for root      | *(Intro, está vacía)*   |
| Switch to unix_socket authentication | `n`                     |
| Change the root password?            | `Y` → contraseña segura |
| Remove anonymous users?              | `Y`                     |
| Disallow root login remotely?        | `Y`                     |
| Remove test database?                | `Y`                     |
| Reload privilege tables now?         | `Y`                     |

### Crear una base de datos y un usuario para la web

No es buena práctica que la web use el usuario `root`, así que creamos uno propio:

```bash
sudo mariadb -u root -p
```

```sql
CREATE DATABASE lamp_db;
CREATE USER 'gabriel'@'localhost' IDENTIFIED BY 'Contraseña_Segura1';
GRANT ALL PRIVILEGES ON lamp_db.* TO 'gabriel'@'localhost';
FLUSH PRIVILEGES;

```

![](/assets/img/ImpAppWeb/2026-10-08-13-17-56-image.png)



---

## 2. Instalación de Apache

```bash
sudo apt install apache2 -y
```

![](/assets/img/ImpAppWeb/2026-10-08-13-18-13-image.png)

Comprobamos el servicio y la versión:

```bash
sudo systemctl status apache2
apache2 -v
```

![](/assets/img/ImpAppWeb/2026-10-08-13-18-53-image.png)



Probamos desde el propio servidor que responde:

```bash
curl -I http://localhost
```

Debe devolver `HTTP/1.1 200 OK`. Desde el navegador del host, entrando en `http://IP_DEL_SERVIDOR`, se verá la página por defecto **"Apache2 Debian Default Page"**.

![](/assets/img/ImpAppWeb/2026-10-08-13-19-14-image.png)

---

## 3. Instalación de PHP y complementos

Debian 13 incluye PHP 8.4. Instalamos PHP, el módulo que lo integra en Apache y las extensiones más habituales:

```bash
sudo apt install php libapache2-mod-php php-mysql php-cli php-common \
                 php-mbstring php-xml php-curl php-gd php-zip php-intl -y
```

| Paquete              | Para qué sirve                                           |
| -------------------- | -------------------------------------------------------- |
| `php`                | Intérprete de PHP                                        |
| `libapache2-mod-php` | Permite que Apache ejecute ficheros `.php`               |
| `php-mysql`          | Conexión de PHP con MariaDB/MySQL (`mysqli`, `PDO`)      |
| `php-cli`            | Ejecutar PHP desde la terminal                           |
| `php-mbstring`       | Manejo de cadenas con caracteres especiales (ñ, tildes…) |
| `php-xml`            | Lectura y escritura de XML                               |
| `php-curl`           | Peticiones HTTP desde PHP                                |
| `php-gd`             | Tratamiento de imágenes                                  |
| `php-zip`            | Ficheros comprimidos                                     |
| `php-intl`           | Internacionalización (fechas, idiomas)                   |

![](/assets/img/ImpAppWeb/2026-10-08-13-19-59-image.png)

Comprobamos la versión y los módulos cargados:

```bash
php -v
php -m | grep -E "mysqli|pdo_mysql|mbstring"
```

![](/assets/img/ImpAppWeb/2026-10-08-13-21-45-image.png)

Hacemos que Apache dé prioridad a `index.php` frente a `index.html`. Editamos:

```bash
sudo nano /etc/apache2/mods-enabled/dir.conf
```

```apache
<IfModule mod_dir.c>
    DirectoryIndex index.php index.html index.cgi index.pl index.xhtml index.htm
</IfModule>
```

Reiniciamos Apache para que cargue el módulo de PHP:

```bash
sudo systemctl restart apache2
```

---

## 4. Comprobación de que todo funciona

### 4.1. PHP funciona en Apache

Creamos un fichero con `phpinfo()`:

```bash
printf "<?php\nphpinfo();\n?>\n" | sudo tee /var/www/html/info.php
```

Accedemos a `http://IP_DEL_SERVIDOR/info.php`. Debe salir la página de información de PHP, donde se ve la versión, el **Server API: Apache 2.0 Handler** y, más abajo, las secciones `mysqli` y `pdo_mysql`.

![](/assets/img/ImpAppWeb/2026-10-08-13-23-37-image.png)

⚠️ **Al terminar la comprobación se borra**, porque muestra mucha información del servidor a cualquiera:

```bash
sudo rm /var/www/html/info.php
```

### 4.2. PHP se conecta a la base de datos

Creamos `/var/www/html/prueba_bd.php`:

```bash
sudo nano /var/www/html/prueba_bd.php
```

```php
<?php
$conexion = new mysqli("localhost", "gabriel", "Contraseña_Segura1", "lamp_db");

if ($conexion->connect_error) {
    die("Error de conexión: " . $conexion->connect_error);
}

echo "<h2>Conexión correcta con MariaDB</h2>";

$resultado = $conexion->query("SELECT nombre, curso FROM alumnos");
while ($fila = $resultado->fetch_assoc()) {
    echo "<p>" . $fila["nombre"] . " - " . $fila["curso"] . "</p>";
}

$conexion->close();
?>
```

Accedemos a `http://IP_DEL_SERVIDOR/prueba_bd.php` y debe mostrar *"Conexión correcta con MariaDB"* y el registro de la tabla `alumnos`. Con esto queda comprobada la pila completa: **Linux → Apache → PHP → MariaDB**.

---

## 5. Sitio web `gabrielmerencioortega.com` y acceso desde el host

### 5.1. Directorio del sitio

```bash
sudo mkdir -p /var/www/gabrielmerencioortega.com
sudo chown -R www-data:www-data /var/www/gabrielmerencioortega.com
sudo chmod -R 755 /var/www/gabrielmerencioortega.com
```

Página principal en PHP (`index.php`):

```bash
sudo nano /var/www/gabrielmerencioortega.com/index.php
```

```php
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Gabriel Merencio Ortega</title>
</head>
<body>
    <h1>Bienvenido a gabrielmerencioortega.com</h1>
    <p>Servidor LAMP - Práctica de 2º ASIR</p>
    <p>Servidor: <?php echo $_SERVER['SERVER_NAME']; ?></p>
    <p>Versión de PHP: <?php echo phpversion(); ?></p>
    <p>Fecha y hora del servidor: <?php echo date("d/m/Y H:i:s"); ?></p>
</body>
</html>
```

### 5.2. VirtualHost

Creamos el fichero de configuración del sitio:

```bash
sudo nano /etc/apache2/sites-available/gabrielmerencioortega.com.conf
```

```apache
<VirtualHost *:80>
    ServerName gabrielmerencioortega.com
    ServerAlias www.gabrielmerencioortega.com
    ServerAdmin webmaster@gabrielmerencioortega.com
    DocumentRoot /var/www/gabrielmerencioortega.com

    <Directory /var/www/gabrielmerencioortega.com>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/gabrielmerencioortega_error.log
    CustomLog ${APACHE_LOG_DIR}/gabrielmerencioortega_access.log combined
</VirtualHost>
```

- `ServerName` / `ServerAlias`: nombres con los que responde el sitio.
- `DocumentRoot`: carpeta de la web.
- `Options -Indexes`: no lista el contenido de carpetas sin índice.
- `ErrorLog` / `CustomLog`: logs propios del sitio, separados de los generales.

Habilitamos el sitio, deshabilitamos el de por defecto, comprobamos la sintaxis y recargamos:

```bash
sudo a2ensite gabrielmerencioortega.com.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

`configtest` debe devolver **`Syntax OK`**.

Comprobamos los sitios activos:

```bash
sudo apache2ctl -S
```

### 5.3. Resolución de nombre en el host

Como el dominio no existe en ningún DNS público, hacemos que el **host** lo resuelva a la IP del servidor editando su fichero `hosts`.

**Host Linux:**

```bash
sudo nano /etc/hosts
```

Añadimos la línea (sustituyendo por la IP real del servidor):

```
172.22.5.223    gabrielmerencioortega.com    www.gabrielmerencioortega.com
```

Comprobamos desde el host:

```bash
ping -c 3 gabrielmerencioortega.com
curl http://gabrielmerencioortega.com
```

Y abrimos en el navegador del host **http://gabrielmerencioortega.com**, que debe mostrar la página de bienvenida con la versión de PHP y la fecha del servidor.

![](/assets/img/ImpAppWeb/2026-10-08-13-27-53-image.png)



![](/assets/img/ImpAppWeb/2026-10-08-13-28-17-image.png)

---

## 6. Logs de acceso, errores y del servicio

### 6.1. Log de acceso

Registra cada petición: IP del cliente, fecha, método, URL, código de respuesta, tamaño y navegador.

```bash
sudo tail -n 20 /var/log/apache2/gabrielmerencioortega_access.log
```

Ver las peticiones en tiempo real mientras se recarga la web desde el host:

```bash
sudo tail -f /var/log/apache2/gabrielmerencioortega_access.log
```



### 6.2. Log de errores

Para que aparezca algún error, pedimos desde el host una página que no existe (`http://gabrielmerencioortega.com/noexiste.php`) y luego:

```bash
sudo tail -n 20 /var/log/apache2/gabrielmerencioortega_error.log
```

También el log de errores general de Apache:

```bash
sudo tail -n 20 /var/log/apache2/error.log
```

Filtrar solo los códigos 404 en el log de acceso:

```bash
sudo grep ' 404 ' /var/log/apache2/gabrielmerencioortega_access.log
```

### 6.3. Logs del servicio (systemd / journald)

Estado y últimas líneas del servicio:

```bash
sudo systemctl status apache2
sudo systemctl status mariadb
```

Historial completo del servicio con `journalctl`:

```bash
sudo journalctl -u apache2 --no-pager -n 30
sudo journalctl -u mariadb --no-pager -n 30
```

Solo los mensajes de hoy:

```bash
sudo journalctl -u apache2 --since today
```

Seguir los mensajes en directo mientras reiniciamos el servicio en otra terminal:

```bash
sudo journalctl -u apache2 -f
```

```bash
sudo systemctl restart apache2
```



---

## Resumen de rutas importantes

| Elemento                          | Ruta                                                            |
| --------------------------------- | --------------------------------------------------------------- |
| Configuración principal de Apache | `/etc/apache2/apache2.conf`                                     |
| Sitios disponibles / habilitados  | `/etc/apache2/sites-available/` · `/etc/apache2/sites-enabled/` |
| VirtualHost de la práctica        | `/etc/apache2/sites-available/gabrielmerencioortega.com.conf`   |
| Raíz de la web                    | `/var/www/gabrielmerencioortega.com/`                           |
| Configuración de PHP para Apache  | `/etc/php/8.4/apache2/php.ini`                                  |
| Configuración de MariaDB          | `/etc/mysql/mariadb.conf.d/50-server.cnf`                       |
| Logs de Apache                    | `/var/log/apache2/`                                             |
| Logs de los servicios             | `journalctl -u apache2` · `journalctl -u mariadb`               |

## Conclusión

Se ha instalado y configurado una pila LAMP completa en Debian 13: MariaDB como gestor de bases de datos con un usuario propio para la web, Apache como servidor web, y PHP 8.4 con las extensiones necesarias. Se ha comprobado que PHP se ejecuta en Apache y que conecta con la base de datos, se ha publicado un sitio propio mediante un VirtualHost accesible desde el host como `gabrielmerencioortega.com`, y se han consultado los logs de acceso, de errores y del servicio.

---
title: "Tarea 1: Introducción a Ansible"
asignatura: ProyIntMod
date: 2026-09-19
resumen: "Primer contacto con Ansible: preparo una VM Debian 13 con acceso por clave SSH y sudo sin contraseña, configuro el inventario hosts.yml y ansible.cfg, y pruebo los módulos ping, command, copy, file, apt, service y user comprobando la idempotencia."
tags:
  - Ansible
  - Debian
  - SSH
  - Automatización
  - Idempotencia
---

En esta práctica vamos a utilizar Ansible por primera vez. Para ello hemos creado previamente una máquina virtual con Debian 13 sin entorno gráfico, que además debía cumplir estos requisitos:

- Tener un usuario sin privilegios con el que podamos acceder a la máquina mediante claves SSH.
- Tener instalado `sudo`, con ese mismo usuario configurado para poder usarlo sin que le pida la contraseña.

Para cumplirlos, hice lo siguiente.

---

## 1. Preparación de la máquina virtual

### 1.1. Acceso por SSH con clave pública/privada

Lo primero es generar el par de claves en el host, es decir, en mi portátil:

```bash
ssh-keygen -t ed25519
```

Una vez creadas, copiamos la clave pública a la máquina virtual:

```bash
ssh-copy-id gabriel@192.168.122.182
```

### 1.2. `sudo` sin contraseña

Ya dentro de la VM que hemos creado, editamos un fichero propio para nuestro usuario:

```bash
sudo visudo -f /etc/sudoers.d/gabriel
```

Este es el contenido del fichero:

```
gabriel ALL=(ALL) NOPASSWD:ALL
```

Utilizamos `visudo` y no un editor directo porque valida la sintaxis antes de guardar. Esto es importante, ya que un error en un fichero de *sudoers* deja el sistema sin `sudo`.

A continuación ajustamos los permisos y comprobamos que todo funciona:

```bash
sudo chmod 0440 /etc/sudoers.d/gabriel
sudo -k
sudo whoami
```

Si todo es correcto, `sudo whoami` debe responder `root` sin pedir nada.

![](/assets/img/ProyIntMod/2026-09-18-23-50-00-image.png)

---

## 2. Configuración de Ansible

Con la máquina ya preparada, vamos a configurar nuestros ficheros de inventario y de configuración. En mi caso se llamarán `hosts.yml` y `ansible.cfg`.

### 2.1. El inventario: `hosts.yml`

En `hosts.yml` hemos puesto lo siguiente:

```yaml
all:
    children:
        servidores:
            hosts:
                debian-ansible:
                    ansible_ssh_host: 192.168.122.181
                    ansible_ssh_user: gabriel
                    ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

![](/assets/img/ProyIntMod/2026-09-18-23-58-34-image.png)

Nuestro inventario se organiza en niveles. Como podemos observar, tenemos varios: `all`, `children`, `servidores`, `hosts` y, por último, el nombre de la máquina.

### 2.2. El fichero de configuración: `ansible.cfg`

El archivo `ansible.cfg` se compone de lo siguiente:

```ini
[defaults]
inventory = hosts.yml
host_key_checking = False
interpreter_python = auto_silent
```

![](/assets/img/ProyIntMod/2026-09-19-00-02-48-image.png)

Este archivo es, básicamente, el fichero de configuración de nuestro Ansible. Lo que hace cada una de estas líneas es lo siguiente:

- `inventory` establece el inventario por defecto, así no hay que escribir `-i hosts.yml` en cada comando.
- `host_key_checking = False` desactiva la comprobación de la huella del host, que en la primera conexión pararía la ejecución esperando un "yes".
- `interpreter_python = auto_silent` deja que Ansible localice por sí solo el intérprete de Python del nodo remoto, sin mostrar avisos. Básicamente, sirve para que la salida del comando que ejecutemos se vea más limpia.

---

## 3. Primeras comprobaciones

### 3.1. Módulo `ping`

Para comprobar que todo funciona, vamos a hacer un ping mediante Ansible con el siguiente comando:

```bash
ansible all -m ping
```

Como podemos ver, he utilizado `all`, pero podemos utilizar lo que deseemos en cualquier momento, por ejemplo solo el nombre de la VM.

![](/assets/img/ProyIntMod/2026-09-19-00-07-50-image.png)

![](/assets/img/ProyIntMod/2026-09-19-00-08-08-image.png)

Como se aprecia, va perfecto, ya que nos responde con el *pong*. Hemos probado los distintos grupos que existen en nuestro `.yml`.

### 3.2. Módulos `command` y `shell`

Vamos a seguir probando comandos. Ahora nos toca utilizar `command`/`shell` para ejecutar la instrucción `hostname`.

![](/assets/img/ProyIntMod/2026-09-19-00-15-16-image.png)

![](/assets/img/ProyIntMod/2026-09-19-00-15-37-image.png)

---

## 4. Copiar un fichero al servidor remoto con `copy`

Crearé un fichero de origen en la carpeta del proyecto de Ansible:

![](/assets/img/ProyIntMod/2026-09-19-14-22-01-image.png)

Ahora vamos a ejecutar el comando de Ansible con `copy` para copiar ese archivo a la VM de forma remota. Lo haremos con este comando:

```bash
ansible debian-ansile -m copy -a "src=prueba.txt dest=/home/gabriel/prueba.txt mode=0644"
```

![](/assets/img/ProyIntMod/2026-09-19-14-25-25-image.png)

Vamos a ejecutarlo de nuevo para ver los cambios que se producen en la salida del comando:

![](/assets/img/ProyIntMod/2026-09-19-14-42-18-image.png)

La primera vez que lo ejecutamos, el fichero no existía en el destino, por eso `copy` lo crea. En la segunda, el módulo calcula el *checksum* del fichero de origen, lo compara con el del fichero que ya está en el nodo remoto, ve que coinciden y no hace nada.

Esa propiedad se llama **idempotencia**: básicamente, ejecutar la misma tarea una vez o cien veces deja el sistema en el mismo estado, porque el módulo comprueba antes los archivos en el nodo y solo actúa si hace falta.

### 4.1. Modificando el fichero de origen

Vamos a modificar el fichero de origen y volveremos a copiarlo:

![](/assets/img/ProyIntMod/2026-09-19-14-45-46-image.png)

![](/assets/img/ProyIntMod/2026-09-19-14-46-10-image.png)

Como vemos, ahora nos da la salida `CHANGED`. Esto se debe a que, al haber modificado el fichero que copiamos, Ansible detecta que algo ha cambiado dentro de él y aplica ese cambio en el destino al ejecutar el comando.

---

## 5. Crear un directorio con `file`

Vamos a crear un directorio de forma remota mediante el módulo `file`:

![](/assets/img/ProyIntMod/2026-09-19-14-53-20-image.png)

Ahora vamos a ver si se ha creado en la VM. Lo comprobaremos aplicando lo que ya hemos aprendido de Ansible y también conectándonos por SSH:

![](/assets/img/ProyIntMod/2026-09-19-14-55-12-image.png)

![](/assets/img/ProyIntMod/2026-09-19-14-54-04-image.png)

---

## 6. Gestión de paquetes y servicios

### 6.1. Instalar nginx con `apt`

Vamos a instalar el paquete nginx con `apt`:

![](/assets/img/ProyIntMod/2026-09-19-15-10-24-image.png)

![](/assets/img/ProyIntMod/2026-09-19-15-10-43-image.png)

Como podemos ver, pasa exactamente igual que antes: una vez está instalado y lo ejecutamos de nuevo, si no hay ningún cambio nos da la salida en verde, ya que no ha realizado ninguna modificación.

### 6.2. Parar el servicio

Ahora pararemos el servicio de nginx mediante Ansible:

![](/assets/img/ProyIntMod/2026-09-19-15-17-27-image.png)

![](/assets/img/ProyIntMod/2026-09-19-15-18-44-image.png)

Como vemos, hemos parado el servicio.

### 6.3. Desinstalar el paquete

Y ahora lo vamos a desinstalar:

![](/assets/img/ProyIntMod/2026-09-19-15-20-34-image.png)

![](/assets/img/ProyIntMod/2026-09-19-15-20-53-image.png)

---

## 7. Gestión de usuarios

Para terminar, crearemos un usuario llamado `pruebas`, verificaremos que existe y lo eliminaremos.

**Creación del usuario:**

![](/assets/img/ProyIntMod/2026-09-19-15-22-18-image.png)

Creado con éxito.

**Comprobamos que existe:**

![](/assets/img/ProyIntMod/2026-09-19-15-22-55-image.png)

**Ahora lo eliminaremos:**

![](/assets/img/ProyIntMod/2026-09-19-15-24-47-image.png)

**Y comprobamos que ya no está.** Este comando he tenido que investigarlo con Claude, ya que no encontraba la forma de poder hacerlo mediante Ansible:

![](/assets/img/ProyIntMod/2026-09-19-15-25-35-image.png)

---

## 8. Conclusión

Con esta primera toma de contacto he visto que Ansible permite administrar una máquina remota sin tener que conectarse a ella y ejecutar los comandos uno a uno. Toda la configuración se reduce a dos ficheros: el inventario `hosts.yml`, donde declaro qué máquinas hay y cómo llegar a ellas, y `ansible.cfg`, que evita tener que repetir opciones en cada comando.

Antes de poder usarlo hay un trabajo previo que resulta imprescindible: el acceso por clave SSH y el `sudo` sin contraseña. Sin esas dos piezas, Ansible se quedaría esperando a que alguien escribiera una contraseña y dejaría de tener sentido automatizar nada.

De todo lo probado, lo que más me ha llamado la atención ha sido la idempotencia. Al repetir la copia del fichero, la instalación de nginx o la creación del directorio, Ansible comprobaba primero el estado del nodo y solo actuaba cuando realmente había algo que cambiar. Se nota en la propia salida del comando: verde cuando no toca nada y `CHANGED` cuando sí. Esto permite volver a lanzar las mismas tareas tantas veces como haga falta sin miedo a romper nada, porque lo que se describe es el estado final que se quiere, no los pasos para llegar a él.

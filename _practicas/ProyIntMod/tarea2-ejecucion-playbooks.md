---
title: "Tarea 2: Ejecución de Playbooks con Ansible"
asignatura: ProyIntMod
date: 2026-09-25
resumen: "Completo y ejecuto un playbook de Ansible que deja listo un servidor Debian 13: actualiza el sistema, instala git y apache2, copia un fichero de configuración y genera la página web desde una plantilla Jinja2. Repaso las variables de nodo y de grupo, los facts, los errores que me fui encontrando y compruebo la idempotencia volviendo a lanzarlo."
tags:
  - Ansible
  - Playbooks
  - Jinja2
  - Idempotencia
  - Apache
---

En esta práctica completamos y ejecutamos un playbook de Ansible que configura un servidor Debian: actualiza el sistema, instala paquetes, copia un fichero de configuración y despliega una página web generada a partir de una plantilla Jinja2.

**Escenario:**

| Máquina | Función | Datos |
|---|---|---|
| Máquina de control | Tiene Ansible instalado | `debian-gabriel` |
| nodo1 | Servidor a configurar (Debian 13 trixie) | `192.168.122.181`, usuario `gabriel` |

---

## 1. Fork y clonación del repositorio

Hacemos un fork de [ejercicios_pi](https://github.com/josedom24/ejercicios_pi) desde GitHub y lo clonamos en la máquina de control:

```bash
git clone https://github.com/SysGabrielMO/ejercicios_pi.git
cd ejercicios_pi/ansible/ejercicio1
```

Estructura del ejercicio:

```
ejercicio1/
├── ansible.cfg        # Configuración de Ansible
├── hosts              # Inventario
├── site.yaml          # Playbook
├── files/foo.conf     # Fichero que se copia tal cual
├── templates/index.j2 # Plantilla Jinja2
└── group_vars/all     # Variables para todos los nodos
```

---

## 2. Inventario y configuración

**ansible.cfg**

```ini
[defaults]
inventory = hosts
host_key_checking = False
interpreter_python = auto_silent
```

- `inventory = hosts`: usa el fichero `hosts` como inventario sin tener que indicar `-i`.
- `host_key_checking = False`: no pide confirmar la huella SSH del nodo.
- `interpreter_python = auto_silent`: detecta Python en el nodo sin mostrar avisos.

**hosts**

```yaml
all:
  children:
    servidores:
      hosts:
        nodo1:
          ansible_ssh_host: 192.168.122.181
          ansible_ssh_user: gabriel
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

Comprobamos que el inventario se lee bien y que hay conexión con el nodo:

```bash
ansible-inventory --graph
ansible all -m ping
```

```
@all:
  |--@ungrouped:
  |--@servidores:
  |  |--nodo1

nodo1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## 3. Variables de nodo y de grupo

Para ver todas las variables que recibe el nodo:

```bash
ansible-inventory --host nodo1
```

```json
{
    "ansible_ssh_host": "192.168.122.181",
    "ansible_ssh_private_key_file": "~/.ssh/id_rsa",
    "ansible_ssh_user": "gabriel",
    "bd_name": "wordpress_bd",
    "bd_pass": "asdasd",
    "bd_user": "wordpress_user"
}
```

- **Variables a nivel de nodo:** `ansible_ssh_host`, `ansible_ssh_user` y `ansible_ssh_private_key_file`. Están definidas en el inventario (`hosts`), dentro de `nodo1`.
- **Variables a nivel de grupo:** `bd_name`, `bd_user` y `bd_pass`. Están definidas en `group_vars/all` y se aplican a todos los nodos.
- **Ficheros consultados:** `hosts` y `group_vars/all`.

---

## 4. Gathering Facts

Los *facts* son variables que Ansible obtiene automáticamente del nodo. Se consultan con el módulo `setup`:

```bash
ansible nodo1 -m setup
ansible nodo1 -m setup -a "filter=ansible_hostname"
ansible nodo1 -m setup -a "filter=ansible_distribution*"
```

Al ejecutar un playbook, esta recogida se hace sola al principio, en la tarea `Gathering Facts`.

---

## 5. El playbook `site.yaml`

- `hosts: all`: las tareas se ejecutan en todos los nodos del inventario.
- `become: true`: las tareas se ejecutan con `sudo`.
- `tasks`: lista de tareas. Cada una tiene un `name` y usa un módulo.

Tareas completadas:

1. **Actualizar el sistema:** el módulo `apt` con `update_cache` y `upgrade`.
2. **Instalar paquetes:** el módulo `apt` con un `loop` que instala `git` y `apache2`.
3. **Copiar fichero:** el módulo `copy` lleva `files/foo.conf` a `/etc/foo.conf`.
4. **Copiar template:** el módulo `template` genera `/var/www/html/index.html` a partir de `index.j2`.

Las tareas de MySQL se han dejado comentadas porque no forman parte del ejercicio y necesitan un servidor de base de datos y PyMySQL en el nodo.

{% raw %}
```yaml
- hosts: all
  become: true
  tasks:
      # Actualizamos paquetes
    - name: Actualizamos el sistema
      apt: update_cache=yes upgrade=yes
      # Instalar paquetes
    - name: "Instalar paquetes con apt"
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - apache2
      # Copia un fichero a la máquina remota
    - name: "Copiar fichero a la máquina remota"
      copy:
        src: files/foo.conf
        dest: /etc/foo.conf
        owner: root
        group: root
        mode: '0644'

      # Copia un template a un fichero
    - name: "Copiar un template a un fichero de la máquina remota"
      template:
        src: templates/index.j2
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: 0644

      # Crear base de datos
    #- name: "Crear base de datos"
    #  community.mysql.mysql_db:
    #    name: "{{ bd_name }}"
    #    state: present

      # Crear usuario de base de datos
    #- name: "Crear usuario de base de datos"
    #  community.mysql.mysql_user:
    #    name: "{{ bd_user }}"
    #    password: "{{ bd_pass }}"
    #    priv: "{{ bd_name }}.*:ALL"
    #    state: present
```
{% endraw %}

**templates/index.j2**

Se cambian las variables `modifica_el_nombre` por las correctas:

{% raw %}
```html
<html lang="es">
<head>
  <meta charset="utf-8">
  <title>Prueba Ansible</title>
</head>

<body>
  <h1>Gathering Facts</h1>
  <p>Este ordenador se llama: {{ ansible_hostname }}</p>
  <p>SO: {{ansible_distribution}} {{ansible_distribution_release}} </p>
  <h1>Variables declaradas por el usuario a nivel de grupo</h1>
  <p>Nombre de la bd: {{ bd_name }}</p>
  <p>Usuario de la bd: {{ bd_user }}</p>
  <h1>Variables declaradas por el usuario a nivel de nodo</h1>
  <p>IP: {{ ansible_ssh_host }}</p>
</body>
</html>
```
{% endraw %}

---

## 6. Ejecución del playbook

```bash
ansible-playbook site.yaml
```

### Errores que me encontré

| Error | Causa | Solución |
|---|---|---|
| `The loop value must resolve to a 'list', not 'str'` | Escribí los paquetes separados por comas | Poner cada paquete en su línea con `-` |
| `No package matching 'mysql-server' is available` | En Debian no existe ese paquete (usa MariaDB) | Instalar solo `git` y `apache2` |
| `dest is required` | La línea `dest:` estaba vacía | `dest: /etc/foo.conf` |
| `Destination directory etc does not exist` | Ruta relativa, sin `/` delante | Usar la ruta absoluta `/etc/foo.conf` |
| `A MySQL module is required` | Las tareas de MySQL necesitan PyMySQL y un servidor de BD | Comentar esas tareas |
| `Failed to parse inventory` | Sangría incorrecta en `hosts` | 2 espacios por nivel, sin tabuladores |

### Ejecución sin errores

Las tareas que hacen cambios aparecen en amarillo (`changed`) y el resumen termina con `failed=0` y `unreachable=0`.

### Segunda ejecución: idempotencia

Al volver a ejecutar el playbook, todas las tareas salen como `ok` y `changed=0`. Ansible comprueba primero el estado del servidor y, como git y apache2 ya están instalados y los ficheros ya son iguales, no vuelve a hacer nada. Esta propiedad se llama **idempotencia**.

### Modificando `foo.conf` en el servidor

```bash
ansible nodo1 -m shell -a "echo 'cambio manual' >> /etc/foo.conf" --become
ansible-playbook site.yaml
```

No se ejecutan todas las tareas: solo la de copiar el fichero aparece como `changed`. Ansible compara el fichero del servidor con el original, ve que son distintos y lo vuelve a copiar. El resto de tareas salen como `ok` porque ya estaban en el estado deseado.

---

## 7. Comprobación

Comprobamos que `foo.conf` está en el servidor:

```bash
ansible nodo1 -m command -a "cat /etc/foo.conf"
```

Y accedemos desde el navegador a `http://192.168.122.181`. La página muestra los tres tipos de variables:

| En la web | Variable | Origen |
|---|---|---|
| debian-ansible | `ansible_hostname` | Facts |
| Debian trixie | `ansible_distribution` + `ansible_distribution_release` | Facts |
| wordpress_bd / wordpress_user | `bd_name` / `bd_user` | `group_vars/all` |
| 192.168.122.181 | `ansible_ssh_host` | Inventario `hosts` |

---

## Repositorio

[github.com/SysGabrielMO/ejercicios_pi](https://github.com/SysGabrielMO/ejercicios_pi)

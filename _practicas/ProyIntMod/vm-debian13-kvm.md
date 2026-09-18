---
title: "Máquina virtual Debian 13 en KVM sin entorno gráfico"
asignatura: ProyIntMod
date: 2026-09-18
resumen: "Creación de una VM Debian 13 con libvirt/KVM, acceso por SSH con clave pública y sudo sin contraseña."
tags:
  - KVM
  - libvirt
  - Debian 13
  - SSH
  - sudo
---

Guía del proceso completo: crear la máquina virtual, acceder por SSH con clave
pública y dejar el usuario con `sudo` sin contraseña.

**Datos de esta instalación:**

| Parámetro | Valor |
|---|---|
| Nombre de la VM | `debian13` |
| IP | `192.168.122.182` |
| Usuario | `gabriel` |
| Red | NAT (`default`) |
| Host | Debian 13 Trixie con libvirt/KVM |

---

## 1. Preparar el host

Instalar libvirt y las herramientas de gestión:

```bash
sudo apt install -y virt-manager virtinst libvirt-daemon-system
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm $USER
```

Después de añadirse a los grupos hay que cerrar sesión y volver a entrar para
que se apliquen.

Descargar la ISO netinst de Debian 13 desde:

```
https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/
```

---

## 2. Crear la máquina virtual

### Opción A — virt-manager (interfaz gráfica)

1. Archivo → Nueva máquina virtual
2. *Local install media* → seleccionar la ISO descargada
3. Memoria: 2048 MB — CPUs: 2
4. Disco: 20 GB
5. Red: **NAT (default)**

### Opción B — virt-install (línea de comandos)

```bash
virt-install \
  --name debian13 \
  --memory 2048 --vcpus 2 \
  --disk size=20,format=qcow2 \
  --cdrom ~/Descargas/debian-13.1.0-amd64-netinst.iso \
  --os-variant debian13 \
  --network network=default \
  --graphics spice
```

Ajustar el nombre exacto del fichero ISO. Si `--os-variant debian13` da error,
usar `debian12` o `generic`.

---

## 3. Durante la instalación

Dos puntos clave:

**Contraseña de root: dejarla en blanco.**
Si se deja vacía, Debian añade automáticamente al primer usuario al grupo
`sudo`. Si se le pone contraseña a root, el usuario no queda en `sudo` y hay
que añadirlo después a mano.

**Selección de software (tasksel): desmarcar todo** con la barra espaciadora y
dejar únicamente:

- `SSH server`
- `Utilidades estándar del sistema`

Sin *Debian desktop environment* ni GNOME: la máquina queda sin entorno
gráfico.

---

## 4. Averiguar la IP de la VM

Con la VM arrancada, desde el host:

```bash
virsh domifaddr debian13
```

Si no devuelve nada:

```bash
virsh net-dhcp-leases default
```

---

## 5. Acceso SSH con clave pública

Si no existe todavía una clave en el host:

```bash
ssh-keygen -t ed25519 -C "gabriel@host"
```

Copiar la clave pública a la VM (pide la contraseña del usuario solo esta vez):

```bash
ssh-copy-id gabriel@192.168.122.182
```

Para forzar una clave concreta si hay varias:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub gabriel@192.168.122.182
```

Comprobar que entra sin pedir contraseña:

```bash
ssh gabriel@192.168.122.182
```

> El acceso por contraseña se deja habilitado: la clave es una vía adicional,
> no un sustituto.

---

## 6. Sudo sin contraseña

Dentro de la VM:

```bash
sudo visudo -f /etc/sudoers.d/gabriel
```

Contenido del fichero, una sola línea:

```
gabriel ALL=(ALL) NOPASSWD:ALL
```

Se usa `visudo` y no un editor directo porque valida la sintaxis antes de
guardar; un error en un fichero de sudoers deja el sistema sin `sudo`.

Ajustar permisos y comprobar:

```bash
sudo chmod 0440 /etc/sudoers.d/gabriel
sudo -k
sudo whoami
```

`sudo whoami` debe responder `root` sin pedir nada.

---

## 7. Atajo de conexión en el host

En `~/.ssh/config` del host:

```
Host debian13
    HostName 192.168.122.182
    User gabriel
    IdentityFile ~/.ssh/id_ed25519
```

A partir de ahí basta con:

```bash
ssh debian13
```

---

## Nota sobre la IP

La red NAT `default` asigna las direcciones por DHCP, así que la IP puede
cambiar al reiniciar la VM. Para fijarla se añade una reserva DHCP por MAC:

```bash
virsh net-edit default
```

Dentro del bloque `<dhcp>`:

```xml
<host mac='52:54:00:xx:xx:xx' name='debian13' ip='192.168.122.182'/>
```

La MAC se obtiene con `virsh domiflist debian13`. Después hay que reiniciar la
red:

```bash
virsh net-destroy default && virsh net-start default
```

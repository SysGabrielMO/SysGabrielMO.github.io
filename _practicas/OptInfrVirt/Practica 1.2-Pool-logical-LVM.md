---
title: "Práctica 1.2. Máquina virtual con pool logical (LVM)"
asignatura: OptInfrVirt
date: 2026-10-09
resumen: "Preparo un grupo de volúmenes LVM dedicado, separado del VG del sistema, y defino sobre él un pool de libvirt de tipo logical. Creo un volumen de 5 GB e instalo una máquina virtual que lo usa directamente como disco vda de tipo bloque. Al final comparo los pools dir y logical: rendimiento y gestión con LVM frente a las ventajas de qcow2."
tags:
  - QEMU/KVM
  - libvirt
  - virsh
  - LVM
  - virt-install
---

**Gabriel Merencio Ortega** · 2º ASIR

En esta práctica optativa del módulo de Infraestructura Virtual creo una máquina virtual cuyo disco principal no es un fichero de imagen (`.qcow2`) guardado en un directorio, sino un **volumen lógico LVM**. Para ello preparo un grupo de volúmenes dedicado, distinto del VG del sistema operativo. Sobre él defino un pool de libvirt de tipo `logical`, creo un volumen de 5 GB e instalo una MV que lo usa como disco `vda`. Al final comparo los pools `dir` y `logical`.

## Enlaces

- [libvirt – Storage Management](https://libvirt.org/storage.html)
- [libvirt – Storage pool and volume XML format](https://libvirt.org/formatstorage.html)
- [virsh(1) – página de manual](https://www.libvirt.org/manpages/virsh.html)
- [virt-install(1) – página de manual](https://manpages.debian.org/virt-install)

## Esquema final

```
Host
└── vg_libvirt_gabriel                 VG LVM independiente del sistema
    └── disco-mv-lvm                   LV de 5 GB
        └── /dev/vg_libvirt_gabriel/disco-mv-lvm
            └── mv-lvm-gabriel         MV que lo usa como vda

libvirt
└── pool_logical_gabriel               Pool de tipo logical sobre vg_libvirt_gabriel
```

## 1. Requisitos previos

Instalo las herramientas de virtualización y LVM en el host:

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst lvm2
```

Compruebo que el servicio de libvirt está en marcha. En versiones recientes es `virtqemud` en lugar de `libvirtd`:

```bash
sudo systemctl status libvirtd
# o bien
sudo systemctl status virtqemud
```

> Todos los comandos `virsh` se ejecutan con `sudo` para trabajar sobre la conexión del sistema (`qemu:///system`), que es donde está el pool `default` y la red `default`.

## 2. Localizar espacio libre para LVM

Necesito un disco o partición libre que **no** pertenezca al VG del sistema. Primero reviso lo que hay:

```bash
lsblk -f
sudo pvs
sudo vgs
sudo lvs
```

- `lsblk -f` muestra discos, particiones, sistemas de ficheros y puntos de montaje.
- `pvs`, `vgs` y `lvs` muestran los volúmenes físicos, grupos de volúmenes y volúmenes lógicos que ya existen.

Compruebo que el disco que voy a usar (en mi caso `/dev/sdb`) no tiene nada montado ni firmas de datos:

```bash
lsblk /dev/sdb
sudo wipefs -n /dev/sdb
```

La opción `-n` solo enseña las firmas; no borra nada.

> ⚠️ `pvcreate`, `vgcreate` y `wipefs` (sin `-n`) destruyen los datos del dispositivo. Hay que comprobar dos veces el nombre del disco antes de seguir.

### Alternativa si no hay disco libre

Si el host no tiene un disco o partición libre, hay dos opciones:

- **MV con virtualización anidada:** una máquina con `--cpu host-passthrough` y un disco extra, dentro de la cual se instala `qemu-kvm`, `libvirt`, `virtinst` y `lvm2` y se repite la práctica.
- **Loop device** (rápido para laboratorio): un fichero del host que se presenta como disco de bloques.

```bash
sudo truncate -s 10G /var/lib/lvm-practica.img
sudo losetup -fP --show /var/lib/lvm-practica.img   # devuelve p. ej. /dev/loop0
```

En ese caso, en los pasos siguientes se usa `/dev/loop0` en lugar de `/dev/sdb`.

**Comprobación:**

```
# pega aquí la salida de lsblk -f y sudo vgs
```

## 3. Crear el grupo de volúmenes dedicado

### 3.1. Volumen físico (PV)

```bash
sudo pvcreate /dev/sdb
sudo pvs
```

`/dev/sdb` debe aparecer como PV, todavía sin VG asignado.

### 3.2. Grupo de volúmenes (VG)

```bash
sudo vgcreate vg_libvirt_gabriel /dev/sdb
sudo vgs
sudo vgdisplay vg_libvirt_gabriel
```

En la columna `VFree` de `vgs` tiene que haber al menos 5 GB libres. Así el almacenamiento de las MVs queda separado del VG del sistema.

**Comprobación:**

```
# pega aquí la salida de sudo vgs
```

## 4. Definir el pool logical en libvirt

Un pool `logical` usa un VG de LVM como origen y trata cada volumen lógico como un disco para las MVs.

Lo defino directamente con `pool-define-as`, sin escribir el XML a mano:

```bash
sudo virsh pool-define-as pool_logical_gabriel logical \
  --source-name vg_libvirt_gabriel \
  --target /dev/vg_libvirt_gabriel
```

Lo arranco y activo el inicio automático:

```bash
sudo virsh pool-start pool_logical_gabriel
sudo virsh pool-autostart pool_logical_gabriel
```

<details>
<summary>Alternativa: definirlo con un fichero XML</summary>

```xml
<pool type='logical'>
  <name>pool_logical_gabriel</name>
  <source>
    <name>vg_libvirt_gabriel</name>
    <format type='lvm2'/>
  </source>
  <target>
    <path>/dev/vg_libvirt_gabriel</path>
  </target>
</pool>
```

```bash
sudo virsh pool-define pool-logical-gabriel.xml
```

</details>

### Comprobación (entregable 1)

```bash
sudo virsh pool-list --all
```

El pool tiene que aparecer como `activo` y con inicio automático `si`:

```
# pega aquí la salida de sudo virsh pool-list --all
```

```bash
sudo virsh pool-dumpxml pool_logical_gabriel
```

```
# pega aquí la salida de sudo virsh pool-dumpxml pool_logical_gabriel
```

Lo importante del XML:

- `type='logical'`: el pool se basa en LVM.
- `<name>vg_libvirt_gabriel</name>` dentro de `<source>`: el VG que usa el pool.
- `<path>/dev/vg_libvirt_gabriel</path>`: el directorio donde LVM expone los volúmenes lógicos como dispositivos de bloques.

## 5. Crear el volumen lógico de la máquina

Creo un volumen de 5 GB dentro del pool. Libvirt crea por debajo el LV correspondiente en `vg_libvirt_gabriel`:

```bash
sudo virsh vol-create-as pool_logical_gabriel disco-mv-lvm 5G --format raw
```

También lo compruebo desde LVM:

```bash
sudo lvs vg_libvirt_gabriel
```

### Comprobación (entregable 2)

```bash
sudo virsh vol-list pool_logical_gabriel
```

Debe aparecer `disco-mv-lvm` con la ruta `/dev/vg_libvirt_gabriel/disco-mv-lvm`:

```
# pega aquí la salida de sudo virsh vol-list pool_logical_gabriel
```

## 6. Instalar la máquina virtual sobre el volumen lógico

Uso una ISO de Debian que tengo en el host (hay que cambiar la ruta por la tuya):

```bash
sudo virt-install \
  --name mv-lvm-gabriel \
  --memory 2048 \
  --vcpus 2 \
  --disk path=/dev/vg_libvirt_gabriel/disco-mv-lvm,format=raw,bus=virtio \
  --cdrom /var/lib/libvirt/images/debian.iso \
  --network network=default,model=virtio \
  --os-variant debian12 \
  --graphics spice
```

Qué hace cada parámetro importante:

- `--name mv-lvm-gabriel`: nombre de la MV.
- `--memory 2048` y `--vcpus 2`: 2 GB de RAM y 2 CPUs virtuales.
- `--disk path=/dev/vg_libvirt_gabriel/disco-mv-lvm`: usa el volumen lógico como disco de la MV.
- `format=raw`: un LV se entrega a QEMU como dispositivo de bloques en crudo, no como `qcow2`.
- `bus=virtio`: disco paravirtualizado; dentro de la MV se verá como `vda`.
- `--cdrom`: ISO de instalación.
- `--network network=default`: red NAT por defecto de libvirt.

> Si la ISO es Debian 13, puedo ver qué variantes reconoce mi sistema con `virt-install --osinfo list | grep debian`. Si no aparece `debian13`, `debian12` funciona igual.

Completo la instalación desde el visor gráfico y, al terminar, quito la ISO si ya no hace falta.

## 7. Comprobar el disco en el XML de la MV

### Comprobación (entregable 3)

```bash
sudo virsh dumpxml mv-lvm-gabriel | grep -A12 -B2 'disco-mv-lvm'
```

```
# pega aquí la salida del comando
```

La configuración es correcta si se cumplen estas dos cosas:

1. El disco es de **tipo bloque**: `<disk type='block' device='disk'>`
2. La fuente apunta al **volumen lógico**: `<source dev='/dev/vg_libvirt_gabriel/disco-mv-lvm'/>`, con `<target dev='vda' bus='virtio'/>`.

Es normal que aparezca otro `<disk type='file' device='cdrom'>`: es la ISO de instalación, no el disco principal.

## 8. Diferencia entre pool `dir` y pool `logical` (entregable 4)

**Pool `dir`**

Un pool `dir`, como `default`, guarda cada disco de máquina virtual como un **fichero** dentro de un directorio del host (por ejemplo `/var/lib/libvirt/images/servidor.qcow2`). Ese fichero vive sobre el sistema de ficheros del host (ext4, xfs…), así que cada escritura de la MV pasa por dos capas: el formato de imagen y el sistema de ficheros del host.

**Pool `logical`**

Un pool `logical` se apoya en un **grupo de volúmenes LVM**. Cada disco de la MV es un **volumen lógico**, que el host expone como un dispositivo de bloques (`/dev/vg_libvirt_gabriel/disco-mv-lvm`). QEMU escribe directamente sobre ese dispositivo, sin fichero de imagen ni sistema de ficheros del host por en medio.

**Ventajas de trabajar a nivel de bloque**

- Hay menos capas entre la MV y el disco real, así que el acceso es más directo y suele rendir mejor en entrada/salida.
- Los discos se gestionan con las herramientas de LVM: `lvextend` para ampliar un disco si hay espacio en el VG, `lvs`/`vgs` para ver el uso, y snapshots de LVM.
- El almacenamiento de las MVs queda separado del sistema de ficheros del host: una MV no puede llenar la partición del sistema.

**Lo que se pierde**

- **Formato qcow2:** el volumen se usa en `raw`, así que no hay aprovisionamiento fino. Los 5 GB se reservan enteros desde el principio aunque la MV solo use 1 GB.
- **Snapshots de qcow2:** desaparecen los snapshots internos de la imagen (`virsh snapshot-create` sobre qcow2). Se pueden hacer snapshots de LVM, pero son otra cosa: necesitan espacio libre en el VG y libvirt no los gestiona igual de cómodo.
- **Compresión y cifrado** propios de qcow2.
- **Portabilidad:** la MV ya no es un fichero que se copia o se mueve sin más. Hay que volcar o clonar el volumen lógico (por ejemplo con `dd` o `virsh vol-download`).

**En resumen:** un pool `dir` con qcow2 es más cómodo para laboratorios, copias y snapshots. Un pool `logical` es mejor cuando importa el rendimiento y se quiere gestionar el espacio con LVM.

## Conclusión

He creado un grupo de volúmenes LVM independiente del sistema, he definido sobre él un pool de libvirt de tipo `logical`, he creado un volumen de 5 GB y he instalado una máquina virtual que lo usa directamente como disco principal. Lo he verificado en el XML del pool (`type='logical'` sobre `vg_libvirt_gabriel`) y en el de la MV, donde el disco `vda` es de tipo `block` y apunta a `/dev/vg_libvirt_gabriel/disco-mv-lvm`.

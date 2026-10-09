---
title: "Práctica 1.2. Máquina virtual con pool logical (LVM)"
asignatura: OptInfrVirt
date: 2026-10-09
resumen: "Como el host no tiene disco libre, creo un loop device de 10 GB y sobre él un grupo de volúmenes LVM dedicado, separado del VG del sistema. Encima defino un pool de libvirt de tipo logical, creo un volumen de 5 GB e instalo una MV Debian 13 que lo usa directamente como disco vda de tipo bloque. Al final comparo los pools dir y logical."
tags:
  - QEMU/KVM
  - libvirt
  - virsh
  - LVM
  - virt-install
---

**Gabriel Merencio Ortega** · 2º ASIR

En esta práctica optativa del módulo de Infraestructura Virtual creo una máquina virtual cuyo disco principal no es un fichero de imagen (`.qcow2`), sino un **volumen lógico LVM**. Como mi host no tiene ningún disco libre, preparo un grupo de volúmenes dedicado sobre un loop device, independiente del VG del sistema. Sobre él defino un pool de libvirt de tipo `logical`, creo un volumen de 5 GB e instalo una MV Debian 13 que lo usa como disco `vda`. Al final comparo los pools `dir` y `logical`.

## Enlaces

- [libvirt – Storage Management](https://libvirt.org/storage.html)
- [libvirt – Storage pool and volume XML format](https://libvirt.org/formatstorage.html)
- [virsh(1) – página de manual](https://www.libvirt.org/manpages/virsh.html)
- [virt-install(1) – página de manual](https://manpages.debian.org/virt-install)

## Esquema final

```
Host (debian-gabriel)
├── vg_debian                          VG del sistema (no se toca)
└── vg_libvirt_gabriel                 VG dedicado sobre /dev/loop0 (10 GB)
    └── disco-mv-lvm                   LV de 5 GB
        └── /dev/vg_libvirt_gabriel/disco-mv-lvm
            └── mv-lvm-gabriel         MV que lo usa como vda

libvirt
└── pool_logical_gabriel               Pool de tipo logical sobre vg_libvirt_gabriel
```

## 1. Requisitos previos

Herramientas de virtualización y LVM en el host:

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst lvm2
```

Todos los comandos `virsh` se lanzan con `sudo` para trabajar sobre la conexión del sistema (`qemu:///system`).

## 2. Comprobar el espacio disponible

Primero reviso los volúmenes LVM que ya existen en el host:

```bash
sudo pvs
sudo vgs
sudo lvs
```

**Comprobación:**

```
  PV             VG        Fmt  Attr PSize    PFree
  /dev/nvme0n1p3 vg_debian lvm2 a--  <952,66g 235,74g

  VG        #PV #LV #SN Attr   VSize    VFree
  vg_debian   1   4   0 wz--n- <952,66g 235,74g

  LV   VG        Attr       LSize
  home vg_debian -wi-ao---- <512,23g
  root vg_debian -wi-ao----  <55,88g
  swap vg_debian -wi-ao----   14,99g
  var  vg_debian -wi-ao---- <133,82g
```

El único VG es `vg_debian`, el del sistema, que **no** se puede usar para la práctica. Además, todo el NVMe pertenece a ese VG, así que no hay ningún disco ni partición libre.

## 3. Crear un disco virtual con un loop device

La solución es crear un fichero de 10 GB y presentarlo al sistema como un dispositivo de bloques mediante un **loop device**:

```bash
sudo fallocate -l 10G /var/lib/libvirt/lvm-practica.img
sudo losetup -fP --show /var/lib/libvirt/lvm-practica.img
```

**Comprobación:**

```
/dev/loop0
```

El fichero queda asociado a `/dev/loop0`, que se comporta como un disco más.

> El loop device no se mantiene tras un reinicio. Para que el pool arranque solo, se puede crear un servicio de systemd que ejecute `losetup` y `vgchange -ay vg_libvirt_gabriel` antes de libvirt.

## 4. Crear el grupo de volúmenes dedicado

Inicializo `/dev/loop0` como volumen físico y creo sobre él un VG exclusivo para libvirt:

```bash
sudo pvcreate /dev/loop0
sudo vgcreate vg_libvirt_gabriel /dev/loop0
sudo pvs
sudo vgs
```

**Comprobación:**

```
  Physical volume "/dev/loop0" successfully created.
  Volume group "vg_libvirt_gabriel" successfully created

  PV             VG                 Fmt  Attr PSize    PFree
  /dev/loop0     vg_libvirt_gabriel lvm2 a--   <10,00g <10,00g
  /dev/nvme0n1p3 vg_debian          lvm2 a--  <952,66g 235,74g

  VG                 #PV #LV #SN Attr   VSize    VFree
  vg_debian            1   4   0 wz--n- <952,66g 235,74g
  vg_libvirt_gabriel   1   0   0 wz--n-  <10,00g <10,00g
```

Ahora hay dos VG: el del sistema y `vg_libvirt_gabriel`, con 10 GB libres solo para las MVs.

## 5. Definir el pool logical en libvirt

Un pool `logical` usa un VG como origen y trata cada volumen lógico como un disco para las MVs. Lo defino, lo arranco y activo el inicio automático:

```bash
sudo virsh pool-define-as pool_logical_gabriel logical \
  --source-name vg_libvirt_gabriel \
  --target /dev/vg_libvirt_gabriel
sudo virsh pool-start pool_logical_gabriel
sudo virsh pool-autostart pool_logical_gabriel
```

**Comprobación (entregable 1):**

```bash
sudo virsh pool-list --all
```

```
 Nombre                 Estado   Inicio automático
----------------------------------------------------
 boot                   activo   si
 default                activo   si
 Descargas              activo   si
 pool_logical_gabriel   activo   si
```

```bash
sudo virsh pool-dumpxml pool_logical_gabriel
```

```xml
<pool type='logical'>
  <name>pool_logical_gabriel</name>
  <uuid>f0363f1e-60ea-4cd1-9747-6d87f7f768f8</uuid>
  <capacity unit='bytes'>10733223936</capacity>
  <allocation unit='bytes'>0</allocation>
  <available unit='bytes'>10733223936</available>
  <source>
    <name>vg_libvirt_gabriel</name>
    <format type='lvm2'/>
  </source>
  <target>
    <path>/dev/vg_libvirt_gabriel</path>
  </target>
</pool>
```

El pool está activo y con inicio automático. Es de tipo `logical`, usa como origen `vg_libvirt_gabriel` y expone los volúmenes en `/dev/vg_libvirt_gabriel`.

## 6. Crear el volumen lógico de la máquina

Creo un volumen de 5 GB dentro del pool. Libvirt crea por debajo el LV en `vg_libvirt_gabriel`:

```bash
sudo virsh vol-create-as pool_logical_gabriel disco-mv-lvm 5G --format raw
```

**Comprobación (entregable 2):**

```bash
sudo virsh vol-list pool_logical_gabriel
```

```
 Nombre         Ruta
------------------------------------------------------
 disco-mv-lvm   /dev/vg_libvirt_gabriel/disco-mv-lvm
```

```bash
sudo lvs vg_libvirt_gabriel
```

```
  LV           VG                 Attr       LSize
  disco-mv-lvm vg_libvirt_gabriel -wi-a----- 5,00g
```

El volumen existe en el pool y es un volumen lógico real de 5 GB dentro del VG.

## 7. Instalar la máquina virtual sobre el volumen lógico

Uso la ISO netinst de Debian 13 que tengo en el directorio de imágenes de libvirt:

```bash
sudo virt-install \
  --name mv-lvm-gabriel \
  --memory 2048 --vcpus 2 \
  --disk path=/dev/vg_libvirt_gabriel/disco-mv-lvm,format=raw,bus=virtio \
  --cdrom /var/lib/libvirt/images/debian-13.7.0-amd64-netinst.iso \
  --network network=default,model=virtio \
  --os-variant debian13 \
  --graphics spice
```

- `--disk path=/dev/vg_libvirt_gabriel/disco-mv-lvm`: el volumen lógico es el disco de la MV.
- `format=raw`: un LV se entrega a QEMU como dispositivo de bloques en crudo, sin qcow2.
- `bus=virtio`: disco paravirtualizado, que dentro de la MV aparece como `vda`.
- `--network network=default`: red NAT por defecto de libvirt.

Durante la instalación elijo el particionado guiado con todo en una partición, y en la selección de programas solo el servidor SSH y las utilidades estándar. Con 5 GB de disco no merece la pena instalar un escritorio.

## 8. Comprobar el disco en el XML de la MV

**Comprobación (entregable 3):**

```bash
sudo virsh dumpxml mv-lvm-gabriel | grep -A12 -B2 'disco-mv-lvm'
```

```xml
<disk type='block' device='disk'>
  <driver name='qemu' type='raw' cache='none' io='native' discard='unmap'/>
  <source dev='/dev/vg_libvirt_gabriel/disco-mv-lvm' index='2'/>
  <backingStore/>
  <target dev='vda' bus='virtio'/>
  <alias name='virtio-disk0'/>
  <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
</disk>
```

El disco principal es de tipo `block` y su origen es el volumen lógico `/dev/vg_libvirt_gabriel/disco-mv-lvm`, no un fichero de imagen. Se presenta a la MV como `vda` por `virtio` y en formato `raw`.

## 9. Diferencia entre pool `dir` y pool `logical` (entregable 4)

Un pool `dir` (como `default`) guarda cada disco como un **fichero** (`.qcow2`) dentro de un directorio del host. Un pool `logical` usa un **grupo de volúmenes LVM**, y cada disco es un **volumen lógico** que la MV usa directamente como dispositivo de bloques (`/dev/vg_libvirt_gabriel/disco-mv-lvm`).

**Ventaja:** trabajar a nivel de bloque quita capas (no hay fichero de imagen ni sistema de ficheros del host por medio), así que el acceso es más directo y rinde mejor. Además, los discos se gestionan con LVM (`lvextend`, `lvs`…).

**Lo que se pierde:** al usar `raw` en lugar de qcow2, el espacio se reserva entero desde el principio. También desaparecen los snapshots, la compresión y el cifrado de qcow2, y la MV deja de ser un fichero fácil de copiar o mover.

## Conclusión

Como el host no tenía disco libre, he creado un loop device de 10 GB y sobre él un grupo de volúmenes independiente del sistema, `vg_libvirt_gabriel`. Encima he definido el pool `pool_logical_gabriel` de tipo `logical`, he creado un volumen de 5 GB y he instalado Debian 13 en una MV que lo usa directamente como disco principal. El XML de la máquina confirma que `vda` es un disco de tipo `block` apuntando a `/dev/vg_libvirt_gabriel/disco-mv-lvm`.

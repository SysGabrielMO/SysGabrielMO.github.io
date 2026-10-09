---
title: "Práctica 1.1 (optativa). Conectar VMs KVM a la red del instituto con un bridge (br0)"
asignatura: OptInfrVirt
date: 2026-10-09
resumen: "Creo un bridge br0 con NetworkManager sobre la tarjeta Ethernet del portátil y añado a las VMs srv-oracle y srv-postgresql una segunda interfaz conectada a él, para que reciban IP del DHCP del instituto y sean accesibles desde el aula sin perder la red NAT default de libvirt. Antes, snapshots de seguridad con virsh."
tags:
  - QEMU/KVM
  - libvirt
  - virsh
  - NetworkManager
  - Bridge
---

**Módulo:** Infraestructura Virtual
**Entorno:** Portátil con Debian 13 (GNOME + NetworkManager) · libvirt/KVM · VMs `srv-oracle` y `srv-postgresql` (Debian 13)

## Objetivo

Las máquinas virtuales `srv-oracle` y `srv-postgresql` tienen que ser accesibles desde la red del instituto. Para ello cada VM tendrá **dos interfaces de red**:

| Interfaz | Red                        | Rango                             | Para qué sirve                                    |
| -------- | -------------------------- | --------------------------------- | ------------------------------------------------- |
| 1        | `default` de libvirt (NAT) | 192.168.122.0/24                  | Acceso desde el propio portátil (también en casa) |
| 2        | Bridge `br0`               | Red del instituto (172.22.0.0/16) | Acceso desde cualquier equipo del instituto       |

---

## 1. Qué cambia con un bridge

La red **default** de libvirt es una red privada (192.168.122.0/24) que solo existe dentro del portátil. El portátil hace de router con **NAT**, así que desde fuera nadie ve las VMs.

Un **bridge (`br0`)** funciona como un **switch virtual** dentro del portátil. A él se conectan la tarjeta de red física y las tarjetas de las VMs. Así las VMs quedan "enchufadas" directamente a la red del instituto, como si fueran otro ordenador más del aula. Piden IP al **DHCP del instituto** y reciben una IP de esa red.

```
RED DEFAULT (NAT):   VM ──> virbr0 (NAT, 192.168.122.x) ──> portátil ──> red instituto

BRIDGE:              VM ───────┐
                               ├── br0 ──> tarjeta física (cable) ──> red instituto (172.22.0.0/16)
                     portátil ─┘
```

> ⚠️ **Importante:** el bridge tiene que hacerse sobre una **tarjeta Ethernet (cable)**. Las tarjetas WiFi no permiten hacer bridge en modo normal, porque el punto de acceso rechaza tramas con MACs que no son la del propio equipo.

---

## 2. Snapshot previa de las VMs

Antes de tocar la red, se hace una snapshot de cada VM para poder volver atrás. Se hace **con las VMs apagadas**, para que las bases de datos queden en un estado limpio.

### 2.1. Apagar las VMs

```bash
sudo virsh shutdown srv-oracle
sudo virsh shutdown srv-postgresql
sudo virsh list --all
```

### 2.2. Comprobar que los discos son qcow2

Las snapshots internas solo funcionan con discos **qcow2**.

```bash
sudo virsh domblklist srv-oracle --details
sudo virsh domblklist srv-postgresql --details
```

```
 Tipo   Dispositivo   Destino   Fuente
--------------------------------------------------------------------------
 file   disk          vda       /var/lib/libvirt/images/debian13-2.qcow2
 file   disk          vdb       /var/lib/libvirt/images/oracle-opt.qcow2
 file   cdrom         sda       -

 Tipo   Dispositivo   Destino   Fuente
------------------------------------------------------------------------------
 file   disk          vda       /var/lib/libvirt/images/srv-postgresql.qcow2
 file   cdrom         sda       -
```

Se confirma el formato real de cada disco (la extensión no lo garantiza):

```bash
sudo qemu-img info /var/lib/libvirt/images/debian13-2.qcow2 | grep "file format"
sudo qemu-img info /var/lib/libvirt/images/oracle-opt.qcow2 | grep "file format"
sudo qemu-img info /var/lib/libvirt/images/srv-postgresql.qcow2 | grep "file format"
```

### 2.3. Crear las snapshots

```bash
sudo virsh snapshot-create-as srv-oracle --name antes-bridge --description "Antes de pasar a br0"
sudo virsh snapshot-create-as srv-postgresql --name antes-bridge --description "Antes de pasar a br0"
```

En `srv-oracle` la snapshot incluye los dos discos (`vda` y `vdb`).

### 2.4. Comprobar y, si hace falta, revertir

```bash
sudo virsh snapshot-list srv-oracle
sudo virsh snapshot-list srv-postgresql

# Solo si algo sale mal:
sudo virsh snapshot-revert srv-oracle antes-bridge
```

La snapshot también guarda la **configuración (XML)** de la VM, así que al revertir la tarjeta de red vuelve a su estado anterior.

---

## 3. Crear el bridge `br0` en el portátil

### 3.1. Localizar la interfaz de cable

```bash
nmcli device status
```

```
DEVICE           TYPE      STATE                   CONNECTION
enx00e04c303c58  ethernet  conectado               Conexión cableada 1
lo               loopback  connected (externally)  lo
virbr0           bridge    connected (externally)  virbr0
wlp2s0           wifi      no disponible           --
...
```

La interfaz de cable es `enx00e04c303c58`. Es un **adaptador Ethernet USB**: el nombre empieza por `enx` seguido de su MAC. Su conexión en NetworkManager es `Conexión cableada 1`.

> El bridge queda asociado a ese nombre, así que hay que usar siempre el mismo adaptador.

### 3.2. Crear el bridge y meter la tarjeta física dentro

```bash
# Crear el bridge (pedirá IP por DHCP)
sudo nmcli con add type bridge ifname br0 con-name br0 ipv4.method auto ipv6.method auto bridge.stp no

# Meter la tarjeta física dentro del bridge
sudo nmcli con add type ethernet ifname enx00e04c303c58 con-name br0-puerto master br0
```

### 3.3. Cambiar de la conexión antigua al bridge

```bash
sudo nmcli con down "Conexión cableada 1"
sudo nmcli con up br0

# Evitar que la conexión antigua se active sola y le "robe" la tarjeta al bridge
sudo nmcli con mod "Conexión cableada 1" connection.autoconnect no
```

> Si `nmcli` dice que la interfaz no está gestionada, hay que revisar que no aparezca en `/etc/network/interfaces`. Si aparece ahí, NetworkManager la ignora.

### 3.4. Comprobar

```bash
ip -br a show br0
ping -c 3 8.8.8.8
```

```
br0              UP             172.22.5.209/16 fe80::40fd:687:2331:d20d/64
```

La IP ahora la tiene **`br0`** y no la tarjeta física. Es lo correcto: el portátil también sale a la red a través del bridge.

> La red del aula resultó ser la **172.22.0.0/16**.

---

## 4. Añadir una segunda interfaz a las VMs conectada a `br0`

No se quita la interfaz de la red default. Se **añade** una segunda tarjeta conectada al bridge.

> Las VMs tienen que estar en `qemu:///system` (el que usan `sudo virsh` y virt-manager por defecto). Las VMs de sesión de usuario no pueden usar bridges.

### 4.1. Añadir la tarjeta (con las VMs apagadas)

Por comando (paquete `virtinst`):

```bash
sudo virt-xml srv-oracle --add-device --network bridge=br0,model=virtio
sudo virt-xml srv-postgresql --add-device --network bridge=br0,model=virtio
```

O desde **virt-manager**: abrir la VM → *Agregar hardware* → *Red* → *Origen de red*: **Dispositivo puente**, nombre `br0`, modelo `virtio`.

### 4.2. Comprobar que cada VM tiene dos interfaces

```bash
sudo virsh domiflist srv-oracle
sudo virsh domiflist srv-postgresql
```

Debe aparecer una interfaz con fuente `default` y otra con fuente `br0`.

---

## 5. Configurar la nueva interfaz dentro de cada VM

### 5.1. Arrancar las VMs y localizar la interfaz nueva

```bash
sudo virsh start srv-oracle
sudo virsh start srv-postgresql
```

Dentro de cada VM (se puede entrar por SSH a su IP 192.168.122.x, que no cambia):

```bash
ip -br a
```

Aparece la interfaz de siempre con su 192.168.122.x y una nueva **sin IP** (por ejemplo `enp7s0`). Esa es la del bridge.

### 5.2. Configurarla por DHCP

En `/etc/network/interfaces` se añade al final (cambiando `enp7s0` por el nombre real):

```
allow-hotplug enp7s0
iface enp7s0 inet dhcp
```

Se levanta y se comprueba:

```bash
sudo ifup enp7s0
ip -br a
ping -c 3 8.8.8.8
```

### 5.3. Rutas por defecto

Como las dos interfaces usan DHCP, la VM puede recibir **dos rutas por defecto**:

```bash
ip route
```

Para el acceso desde el instituto no afecta: la red 172.22.0.0/16 está conectada directamente a la interfaz del bridge, así que los equipos del instituto llegan a la VM igualmente. Solo cambia por qué interfaz sale la VM a internet.

---

## 6. Resultado

| VM               | Interfaz red default (NAT) | Interfaz bridge `br0` |
| ---------------- | -------------------------- | --------------------- |
| `srv-oracle`     | 192.168.122.x              | 172.22.x.x            |
| `srv-postgresql` | 192.168.122.20             | 172.22.5.211          |

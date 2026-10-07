---
title: "Práctica 1. QEMU/KVM + libvirt"
asignatura: OptInfrVirt
date: 2026-10-07
resumen: "Monto un escenario virtualizado entero desde la línea de comandos con QEMU/KVM y libvirt: un router Debian que da salida a una red muy aislada con SNAT y publica la web con DNAT, un NAS Alpine con un disco extra compartido por NFS y un servidorWeb Ubuntu creado por clonación enlazada y cloud-init que sirve con nginx la página alojada en el NAS. Incluye también redimensionado de disco en caliente y snapshots con virsh."
tags:
  - QEMU/KVM
  - libvirt
  - virsh
  - NFS
  - nftables
---

**Gabriel Merencio Ortega** · 2º ASIR

En esta práctica vamos a crear un escenario virtualizado con QEMU/KVM + libvirt en un servidor sin entorno gráfico, usando solo la línea de comandos (`virsh`, `virt-install` y `qemu-img`). El escenario tiene un router, un servidor web y un servidor NAS, y el servidor web sirve una página estática desde un directorio que el NAS comparte por NFS.

## Enlaces

- Enunciado: [Práctica: QEMU/KVM + libvirt](https://fp.josedomingo.org/iv/main/u1/practica/)

## Objetivos

- Crear un escenario virtualizado con QEMU/KVM + libvirt en un servidor sin entorno gráfico.
- Hacerlo todo desde la línea de comandos con `virsh`, `virt-install` y `qemu-img` (sin `virt-manager`).
- Montar un servidor web que sirve contenido estático desde un directorio compartido por un NAS.

## Esquema del escenario

```
   Host (exterior)
        │
  RED NAT default ── 192.168.122.0/24
        │
   ┌─────────┐ enp1s0: 192.168.122.161 (DHCP con reserva)
   │ router  │
   └─────────┘ enp2s0: 192.168.200.1
        │
  RED MUY AISLADA red_intra ── 192.168.200.0/24
        │
   ├── servidorWeb  192.168.200.10  (Ubuntu 26.04, clon enlazado + cloud-init)
   └── servidorNAS  192.168.200.20  (Alpine 3.24, instalado desde ISO)
```

| Máquina | Sistema | Instalación | Redes | Hostname |
|---|---|---|---|---|
| router | Debian 13 | Por red (`--location`) | default + red_intra | router-gabriel |
| servidorNAS | Alpine 3.24 | Desde ISO (`--cdrom`) | red_intra | nas-gabriel |
| servidorWeb | Ubuntu 26.04 | Clonación enlazada + cloud-init | red_intra | web-gabriel |

Como `red_intra` es **muy aislada** (sin DHCP, sin IP en el host), todas las IPs, puertas de enlace y DNS de esa red se configuran a mano. Como DNS uso `192.168.122.1`, el dnsmasq que libvirt ofrece en la red `default`, al que se llega a través del SNAT del router.

---

## Parte 1: Infraestructura

### Preparación del host

Para que `virsh` trabaje siempre contra el hipervisor del sistema (y no contra `qemu:///session`):

```bash
printf "export LIBVIRT_DEFAULT_URI='qemu:///system'\n" >> ~/.bashrc
source ~/.bashrc
virsh uri
```

### Red `red_intra` (muy aislada)

Una red sin `<forward>` ni `<ip>` es solo un switch virtual: el host no tiene IP en ella.

`red_intra.xml`:

```xml
<network>
  <name>red_intra</name>
  <bridge name='virbr10'/>
</network>
```

```bash
virsh net-define red_intra.xml
virsh net-start red_intra
virsh net-autostart red_intra
```

**Comprobación:** las dos redes de la práctica (`default` y `red_intra`) activas, persistentes y con inicio automático.

```
$ virsh net-list --all
 Nombre      Estado   Inicio automático   Persistente
-------------------------------------------------------
 br-nat      activo   si                  si
 br-red1     activo   si                  si
 br-red2     activo   si                  si
 default     activo   si                  si
 red_intra   activo   si                  si
```

### Router: Debian 13 instalado por red

`virt-install` descarga el kernel y el initrd del repositorio de Debian y el instalador baja el resto de paquetes. Como no hay entorno gráfico, la instalación se hace por consola serie. El `--extra-args` con `---` hace que el parámetro de consola se copie también al sistema instalado.

```bash
virt-install --connect qemu:///system \
             --virt-type kvm \
             --name router \
             --location http://deb.debian.org/debian/dists/trixie/main/installer-amd64/ \
             --os-variant debian13 \
             --disk size=10 \
             --memory 1024 \
             --vcpus 1 \
             --network network=default \
             --network network=red_intra \
             --graphics none --console pty,target_type=serial \
             --extra-args="console=ttyS0,115200n8 --- console=ttyS0,115200n8"
```

En el instalador: interfaz principal `enp1s0` (la de `default`), hostname `router-gabriel`, root **sin contraseña** (así el primer usuario entra en el grupo `sudo`), usuario `user` y solo *SSH server* + utilidades estándar. Por consola serie el instalador solo ofrece inglés: idioma `English`, ubicación `other > Europe > Spain`, locale `en_US.UTF-8` y teclado `Spanish`.

Autoarranque y reserva DHCP para que el router tenga siempre la misma IP (importante para SSH y para el DNAT):

```bash
virsh autostart router
virsh net-update default add ip-dhcp-host "<host mac='52:54:00:b7:c3:ec' ip='192.168.122.161'/>" --live --config
```

**sudo sin contraseña** para `user`:

```bash
printf 'user ALL=(ALL) NOPASSWD:ALL\n' | sudo tee /etc/sudoers.d/user
sudo chmod 440 /etc/sudoers.d/user
sudo visudo -c
```

**Segunda interfaz** (`/etc/network/interfaces`, añadido al final):

```
auto enp2s0
iface enp2s0 inet static
    address 192.168.200.1/24
```

```bash
sudo ifup enp2s0
```

**Comprobación:** acceso por SSH sin contraseña, hostname y `sudo` sin contraseña. El `-n` hace que `sudo` falle en vez de pedir contraseña, así que si responde `root` queda demostrado.

```
$ ssh router-kvm hostname
router-gabriel
$ ssh router-kvm sudo -n whoami
root
$ ssh router-kvm sudo -l
User user may run the following commands on router-gabriel:
    (ALL) NOPASSWD: ALL
```

### SNAT en el router (persistente)

Reenvío de paquetes activado de forma permanente:

```bash
printf 'net.ipv4.ip_forward=1\n' | sudo tee /etc/sysctl.d/99-ipforward.conf
sudo sysctl --system
```

Regla SNAT con **nftables** en `/etc/nftables.conf`. Uso `masquerade` porque la IP del router en `default` llega por DHCP: es un SNAT que siempre usa la IP actual de la interfaz de salida.

```
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "enp1s0" ip saddr 192.168.200.0/24 masquerade
    }
}
```

```bash
sudo systemctl enable --now nftables
sudo nft list ruleset
```

> Las reglas están escritas en formato nativo de nftables, así que se consultan con `nft list ruleset`, no con `iptables -L`.

**Comprobación:** la regla y el reenvío siguen activos después de reiniciar el router (configuración persistente), y el router sale a Internet.

```
$ ssh router-kvm 'sudo nft list table ip nat; sudo sysctl net.ipv4.ip_forward'
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "enp1s0" ip saddr 192.168.200.0/24 masquerade
    }
}
net.ipv4.ip_forward = 1

$ ssh router-kvm ping -c 2 1.1.1.1
2 packets transmitted, 2 received, 0% packet loss
```

### servidorNAS: Alpine 3.24 desde ISO

```bash
cd /var/lib/libvirt/images
sudo wget https://dl-cdn.alpinelinux.org/alpine/v3.24/releases/x86_64/alpine-virt-3.24.2-x86_64.iso

virt-install --connect qemu:///system \
             --virt-type kvm \
             --name servidorNAS \
             --cdrom /var/lib/libvirt/images/alpine-virt-3.24.2-x86_64.iso \
             --os-variant alpinelinux3.21 \
             --disk size=5 \
             --memory 512 \
             --vcpus 1 \
             --network network=red_intra \
             --graphics none --console pty,target_type=serial
```

Dentro, con `setup-alpine`: teclado `es`, hostname `nas-gabriel`, IP `192.168.200.20/24`, gateway `192.168.200.1`, DNS `192.168.122.1`, usuario `user`, SSH `openssh`, disco `vda` en modo `sys`.

```bash
virsh autostart servidorNAS
```

`/etc/network/interfaces` del NAS:

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 192.168.200.20/24
        gateway 192.168.200.1
```

**Comprobación:** el NAS sale a Internet a través del router (su ruta por defecto es `192.168.200.1`).

```
$ ssh nas-kvm ping -c 2 1.1.1.1
2 packets transmitted, 2 packets received, 0% packet loss
$ ssh nas-kvm ip route
default via 192.168.200.1 dev eth0  metric 1 onlink
192.168.200.0/24 dev eth0 scope link  src 192.168.200.20
```

#### Disco extra de 1 GB (qcow2)

Creado y conectado en caliente con `virsh`:

```bash
virsh vol-create-as default nas-data.qcow2 1G --format qcow2
virsh attach-disk servidorNAS /var/lib/libvirt/images/nas-data.qcow2 vdb \
      --driver qemu --subdriver qcow2 --targetbus virtio --persistent
```

Dentro del NAS, formateado **sin particiones** (así luego ampliarlo es más sencillo) y montado por UUID en `/etc/fstab`:

```bash
mkfs.ext4 /dev/vdb
mkdir -p /srv/data
blkid /dev/vdb
printf 'UUID=c14d68ec-fd1f-4124-a5b2-184010adcb1b  /srv/data  ext4  defaults  0  2\n' >> /etc/fstab
mount -a
```

**Comprobación:** el disco está montado en `/srv/data` y sigue montado después de reiniciar, gracias a la línea de `fstab`.

```
$ ssh nas-kvm df -h /srv/data
Filesystem                Size      Used Available Use% Mounted on
/dev/vdb                973.4M    280.0K    905.9M   0% /srv/data
$ ssh nas-kvm grep /srv/data /etc/fstab
UUID=c14d68ec-fd1f-4124-a5b2-184010adcb1b  /srv/data  ext4  defaults  0  2
```

### servidorWeb: Ubuntu 26.04 con clonación enlazada y cloud-init

Clon enlazado creado con `virsh` a partir de la imagen cloud, que queda como *backing file* de solo lectura:

```bash
virsh vol-create-as default servidorWeb.qcow2 10G --format qcow2 \
      --backing-vol ubuntu-26.04-server-cloudimg-amd64.qcow2 --backing-vol-format qcow2
sudo qemu-img info /var/lib/libvirt/images/servidorWeb.qcow2
```

```
virtual size: 10 GiB (10737418240 bytes)
disk size: 196 KiB
backing file: /var/lib/libvirt/images/ubuntu-26.04-server-cloudimg-amd64.qcow2
backing file format: qcow2
```

`user-data.yaml`:

```yaml
#cloud-config
hostname: web-gabriel
manage_etc_hosts: true
users:
  - name: user
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
    plain_text_passwd: ********
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... gabriel@debian-gabriel
      - ssh-rsa AAAA... jose@debian
```

`network-config.yaml`:

```yaml
version: 2
ethernets:
  red_intra:
    match:
      name: "en*"
    addresses:
      - 192.168.200.10/24
    routes:
      - to: default
        via: 192.168.200.1
    nameservers:
      addresses:
        - 192.168.122.1
```

```bash
virt-install --connect qemu:///system \
             --virt-type kvm \
             --name servidorWeb \
             --os-variant ubuntu25.10 \
             --memory 1024 \
             --vcpus 1 \
             --disk vol=default/servidorWeb.qcow2 \
             --import \
             --network network=red_intra \
             --graphics none --console pty,target_type=serial \
             --cloud-init user-data=user-data.yaml,network-config=network-config.yaml

virsh autostart servidorWeb
```

**Comprobación:** el servidorWeb sale a Internet a través del router.

```
$ ssh web-kvm ping -c 2 1.1.1.1
2 packets transmitted, 2 received, 0% packet loss
$ ssh web-kvm ip route
default via 192.168.200.1 dev enp1s0 proto static
192.168.200.0/24 dev enp1s0 proto kernel scope link src 192.168.200.10
```

**Comprobación:** las tres máquinas se inician con el host.

```
$ virsh list --all --autostart
 Id   Nombre        Estado
--------------------------------
 1    router        ejecutando
 6    servidorNAS   ejecutando
 7    servidorWeb   ejecutando
```

### Acceso SSH a través del router (ProxyJump)

El host no tiene IP en `red_intra`, así que a las máquinas internas se llega saltando por el router. `~/.ssh/config`:

```
Host router-kvm
    HostName 192.168.122.161
    User user

Host nas-kvm
    HostName 192.168.200.20
    User user
    ProxyJump router-kvm

Host web-kvm
    HostName 192.168.200.10
    User user
    ProxyJump router-kvm
```

Con ProxyJump la sesión SSH se negocia directamente desde el host a través de un túnel por el router, así que la clave privada nunca sale del host (a diferencia de `ssh -A`).

Claves: la mía con `ssh-copy-id` y la del profesor añadida a `authorized_keys` en las tres máquinas.

**Comprobación:** acceso sin contraseña y hostname del NAS y del servidorWeb.

```
$ ssh nas-kvm hostname
nas-gabriel
$ ssh web-kvm hostname
web-gabriel
```

### Pregunta: ventaja de la clonación enlazada

> La gran ventaja es la **rapidez y el ahorro de espacio**. La instalación por red y la de ISO obligan a pasar por todo el instalador y cada máquina acaba con un disco completo. Con la clonación enlazada no se instala nada: se crea un disco nuevo que usa la imagen cloud como base de solo lectura y solo guarda los cambios. El disco de `servidorWeb` ocupaba 196 KiB frente a los 825 MiB de la base. Además, con cloud-init la máquina se configura sola en el primer arranque.

---

## Parte 2: Instalación de servicios

### nginx en el servidorWeb

```bash
sudo apt update
sudo apt install -y nginx
```

### DNAT en el router

Se añade la cadena `prerouting` a la tabla `ip nat`: el destino se cambia **antes** de la decisión de enrutamiento.

```
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        iifname "enp1s0" tcp dport 80 dnat to 192.168.200.10
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "enp1s0" ip saddr 192.168.200.0/24 masquerade
    }
}
```

```bash
sudo nft -f /etc/nftables.conf
```

> El DNAT hay que probarlo desde fuera del escenario (el host). Desde el propio router no funciona: el tráfico que genera el router pasa por `output`, no por `prerouting`.

**Comprobación:** desde el host (exterior del escenario), el puerto 80 del router responde y quien contesta es el nginx del servidorWeb.

```
$ curl -I http://192.168.122.161
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Content-Type: text/html
```

### Servidor NFS en el servidorNAS

```bash
sudo apk add nfs-utils
printf '/srv/data 192.168.200.10(ro,sync,no_subtree_check)\n' | sudo tee /etc/exports
sudo rc-update add nfs
sudo rc-service nfs start
sudo exportfs -ra
```

Se comparte solo con la IP del servidorWeb y en solo lectura: el contenido se edita siempre en el NAS.

En `/srv/data/index.html` está la página estática con mi nombre y la fecha.

**Comprobación:** el recurso está compartido solo con el servidorWeb y en solo lectura.

```
$ ssh nas-kvm sudo exportfs -v
/srv/data         192.168.200.10(sync,wdelay,hide,no_subtree_check,sec=sys,ro,secure,root_squash,no_all_squash)
```

### Montaje NFS en el servidorWeb

```bash
sudo apt install -y nfs-common
sudo mkdir -p /var/www/data
showmount -e 192.168.200.20
printf '192.168.200.20:/srv/data  /var/www/data  nfs  ro,_netdev,nofail,x-systemd.automount  0  0\n' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
```

Las opciones evitan problemas de orden al arrancar el host (las tres máquinas se encienden a la vez): `_netdev` espera a la red, `nofail` no bloquea el arranque si el NAS aún no responde y `x-systemd.automount` monta en el primer acceso.

**Comprobación:** el directorio del NAS está montado en `/var/www/data` y el montaje es persistente.

```
$ ssh web-kvm 'ls /var/www/data; df -h /var/www/data; grep nfs /etc/fstab'
index.html
lost+found
Filesystem                Size  Used Avail Use% Mounted on
192.168.200.20:/srv/data  974M  256K  906M   1% /var/www/data
192.168.200.20:/srv/data  /var/www/data  nfs  ro,_netdev,nofail,x-systemd.automount  0  0
```

### Virtual Host `data.gabriel.org`

`/etc/nginx/sites-available/data.gabriel.org`:

```nginx
server {
    listen 80;
    server_name data.gabriel.org;

    root /var/www/data;
    index index.html;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/data.gabriel.org /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

En el host, para resolver el nombre hacia el router:

```bash
printf '192.168.122.161  data.gabriel.org\n' | sudo tee -a /etc/hosts
```

> Si se accede por IP se ve la página por defecto de nginx: el Virtual Host se elige por el **nombre** que pide el navegador.

**Comprobación:** acceso a la página desde el exterior del escenario, con mi nombre y la fecha.

```
$ curl http://data.gabriel.org
...
    <h1>Practica QEMU/KVM + libvirt</h1>
    <p>Servido desde el servidorNAS por NFS</p>
    <p><strong>Gabriel Merencio Ortega</strong></p>
    <p>6 de octubre de 2026</p>
...
```

### Pregunta: ¿por qué NFS en vez de copiar los ficheros?

> Porque así la web está guardada en un solo sitio, el NAS, y el servidorWeb solo la sirve. Si copiáramos los ficheros tendríamos dos copias y habría que actualizarlas a mano cada vez. Si cambiamos la página en el NAS, el cambio se ve al instante en la web, sin copiar nada ni reiniciar nginx.

---

## Parte 3: Otras operaciones

### Redimensionar el disco del NAS a 2 GB

En caliente, sin apagar la máquina:

```bash
virsh blockresize servidorNAS vdb 2G
```

Dentro del NAS (en Alpine, `resize2fs` está en `e2fsprogs-extra`). Como `vdb` no tiene particiones, basta con ampliar el sistema de ficheros:

```bash
sudo apk add e2fsprogs-extra
sudo resize2fs /dev/vdb
```

**Comprobación:** el disco y el sistema de ficheros miden 2 GB. El servidorWeb ve el nuevo tamaño a través del NFS sin tocar nada.

```
$ virsh domblkinfo servidorNAS vdb
Capacidad:      2147483648
$ ssh nas-kvm df -h /srv/data
/dev/vdb                  1.9G    284.0K      1.8G   0% /srv/data
$ ssh web-kvm df -h /var/www/data
192.168.200.20:/srv/data  2.0G  256K  1.9G   1% /var/www/data
```

### Snapshots

```bash
virsh snapshot-create-as router --name final --description "Escenario terminado"
virsh snapshot-create-as servidorNAS --name final --description "Escenario terminado"
virsh snapshot-create-as servidorWeb --name final --description "Escenario terminado"
```

Para volver a ese estado: `virsh snapshot-revert <máquina> final`.

**Comprobación:** lista de snapshots de cada máquina.

```
$ virsh snapshot-list router
 Nombre   Hora de creación            Estado
-----------------------------------------------
 final    2026-10-07 10:00:10 +0200   running

$ virsh snapshot-list servidorNAS
 final    2026-10-07 10:00:22 +0200   running

$ virsh snapshot-list servidorWeb
 final    2026-10-07 10:00:31 +0200   running
```

### Pregunta: ¿cuándo hacer el snapshot?

> Conviene hacerlo **después** de instalar y configurar los servicios, cuando todo funciona y está comprobado. Un snapshot sirve para volver a un estado bueno conocido. Si lo hiciéramos antes, al restaurarlo perderíamos nginx, el NFS, el DNAT y toda la configuración, y tendríamos que repetirla.

---

## Problemas que me encontré (y cómo los resolví)

- **`virsh` no encontraba las máquinas**: estaba conectado a `qemu:///session` y las máquinas están en `qemu:///system`. Solución: `LIBVIRT_DEFAULT_URI` en el `.bashrc`.
- **Alpine no arrancaba tras instalar**: en `setup-alpine`, el disco por defecto es `[none]`. Al pulsar Enter se configuró el sistema *live* pero no se escribió nada en el disco (`virsh domblkinfo` mostraba ~1 MB ocupado). Hay que escribir `vda` y elegir `sys`.
- **El NAS no tenía red**: `red_intra` no tiene DHCP; hay que poner IP, gateway y DNS a mano.
- **`sudo` no existe en Alpine**: está en el repositorio `community`, que hay que activar en `/etc/apk/repositories`. Recordatorio: Alpine usa `apk` y OpenRC (`rc-service`, `rc-update`).
- **SSH conectaba a una IP que no era**: había bloques `Host router` y `Host web` antiguos en `~/.ssh/config`; SSH usa el primero que coincide. Solución: nombres propios (`router-kvm`, `nas-kvm`, `web-kvm`).
- **El DNAT "no funcionaba"**: lo estaba probando desde el propio router. Hay que probarlo desde fuera del escenario.

## Conclusión

Con libvirt se crean las máquinas y las redes; el router conecta la red aislada con el exterior usando SNAT (salida a Internet) y DNAT (acceso a la web); el NAS guarda los datos y los comparte por NFS; y el servidorWeb los publica con nginx. Además, he usado las tres formas de crear máquinas virtuales (por red, desde ISO y por clonación enlazada con cloud-init) y operaciones habituales como redimensionar discos en caliente y hacer snapshots, todo desde `virsh`.

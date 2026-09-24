---
title: "Escenario de redes con virt-manager: router, servidor web y clientes"
asignatura: Servicios
date: 2026-09-24
resumen: "Monto en virt-manager un escenario con tres redes virtuales (NAT, aislada y muy aislada) y cuatro máquinas: router Debian, servidor web Ubuntu, cliente Fedora y cliente Windows 11. Configuro el direccionamiento estático, los FQDN, el acceso por SSH con clave y ProxyJump, y doy salida a internet a las redes internas con SNAT, publicando Apache al exterior mediante DNAT."
tags:
  - Redes
  - virt-manager
  - iptables
  - SSH
  - Apache
---

## Parte 1: Configuración con direccionamiento estático

Dominio usado en todo el escenario: `gabrielmerencio.org`

![Esquema del escenario](/assets/img/Servicios/esquema.png)

| Máquina | Sistema | FQDN |
|---|---|---|
| router | Debian 13 (trixie), sin entorno gráfico | `router.gabrielmerencio.org` |
| Servidor Web | Ubuntu Server | `web.gabrielmerencio.org` |
| cliente1 | Fedora (sin entorno gráfico) | `cliente1.gabrielmerencio.org` |
| cliente2 | Windows 11 | `cliente2.gabrielmerencio.org` |

---

## 0. Redes virtuales

| Red | Bridge | Tipo | Direccionamiento | DHCP | Máquinas |
|---|---|---|---|---|---|
| br-nat | `br-nat` | NAT | 192.168.101.0/24 (host .1) | No | router |
| br-red1 | `br-red1` | Aislada | 192.168.102.0/24 (host .1) | No | router, web |
| br-red2 | `br-red2` | Muy aislada | 172.20.0.0/16 (host sin IP) | No | router, cliente1, cliente2 |

**Diferencia entre aislada y muy aislada:** en la red aislada el host tiene una IP dentro de la red (puede hablar con las VMs); en la muy aislada el host no tiene ninguna IP, así que solo las VMs se ven entre ellas.

### XML final de las redes

**br-nat**
```xml
<network>
  <name>br-nat</name>
  <forward mode="nat">
    <nat>
      <port start="1024" end="65535"/>
    </nat>
  </forward>
  <bridge name="br-nat" stp="on" delay="0"/>
  <domain name="br-nat"/>
  <ip address="192.168.101.1" netmask="255.255.255.0">
  </ip>
</network>
```

**br-red1**
```xml
<network>
  <name>br-red1</name>
  <bridge name="br-red1" stp="on" delay="0"/>
  <domain name="br-red1"/>
  <ip address="192.168.102.1" netmask="255.255.255.0">
  </ip>
</network>
```

**br-red2**
```xml
<network>
  <name>br-red2</name>
  <bridge name="br-red2" stp="on" delay="0"/>
  <domain name="red-muy-aislada"/>
</network>
```

### Cambios realizados sobre las redes que ya tenía

1. **Nombre del bridge.** Las redes se llamaban `br-nat`, `br-red1` y `br-red2`, pero sus bridges eran `virbr1`, `virbr2` y `virbr3`. El enunciado pide que el **bridge** tenga ese nombre, así que se cambió la línea `<bridge name="...">`.
2. **DHCP en br-nat.** Se eliminó el bloque `<dhcp>`, porque el enunciado indica que no debe tener. Además, en esa red solo está el router, y un router debe tener siempre IP fija.

**Forma gráfica:**
1. *Editar → Preferencias → General →* activar **Habilitar edición XML**.
2. *Editar → Detalles de la conexión → Redes virtuales.*
3. Seleccionar la red, **detenerla** (si está activa, virt-manager muestra la configuración en ejecución y parece que el cambio no se guarda).
4. Pestaña **XML**, editar, **Aplicar**, e **iniciar** la red.

**Forma por terminal:**
```bash
virsh -c qemu:///system net-destroy br-nat
virsh -c qemu:///system net-edit br-nat
virsh -c qemu:///system net-start br-nat
```

**Comprobación:**
```bash
ip -br a | grep br-
```
```
br-nat           UP             192.168.101.1/24
br-red1          UP             192.168.102.1/24
br-red2          UP
```

Los bridges aparecen en `DOWN` hasta que se conecta alguna VM; es normal.

### Solapamiento de redes: cambio de br-red2 a 172.20.0.0/16

Al principio usé `172.22.0.0/16` para la red muy aislada, pero el adaptador de red físico del portátil estaba en la red real `172.22.0.0/16` (`172.22.0.155/16`). Tener la misma red dentro y fuera provoca conflictos de rutas y posibles IPs duplicadas, así que se cambió a **`172.20.0.0/16`**, que sigue cumpliendo la máscara /16.

---

## 1. Configuración de las interfaces de red

### Plan de direcciones

| Máquina | Interfaz → red | IP | Puerta de enlace | DNS |
|---|---|---|---|---|
| router | enp1s0 → br-nat | 192.168.101.2/24 | 192.168.101.1 | 192.168.101.1 |
| router | enp2s0 → br-red1 | 192.168.102.254/24 | — | — |
| router | enp3s0 → br-red2 | 172.20.0.1/16 | — | — |
| web | enp1s0 → br-red1 | 192.168.102.2/24 | 192.168.102.254 | 192.168.101.1 |
| cliente1 | enp1s0 → br-red2 | 172.20.0.2/16 | 172.20.0.1 | 192.168.101.1 |
| cliente2 | Ethernet → br-red2 | 172.20.0.3/16 | 172.20.0.1 | 192.168.101.1 |

- En br-red1 el router usa la **.254** porque la .1 ya la tiene el host.
- En br-red2 el host no tiene IP, así que el router usa la **.1**.
- El DNS es `192.168.101.1` por el motivo explicado en el apartado 5.

### Router (Debian)

Para saber qué interfaz va a cada red, se comparan las MAC de `ip -br link` con las de cada NIC en los detalles de la VM en virt-manager.

Comprobar qué gestor de red se usa:
```bash
systemctl is-enabled networking
systemctl is-enabled systemd-networkd
systemctl is-enabled NetworkManager
```
Usa `networking` (ifupdown), así que se configura `/etc/network/interfaces`:

```
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto enp1s0
iface enp1s0 inet static
    address 192.168.101.2/24
    gateway 192.168.101.1

auto enp2s0
iface enp2s0 inet static
    address 192.168.102.254/24

auto enp3s0
iface enp3s0 inet static
    address 172.20.0.1/16
```

Solo la interfaz que va hacia internet lleva `gateway`.

```bash
sudo systemctl restart networking
```

**IP forwarding** (para que el router reenvíe paquetes entre redes), de forma permanente:
```bash
printf "net.ipv4.ip_forward=1\n" | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
cat /proc/sys/net/ipv4/ip_forward
```
```
1
```

### Servidor Web (Ubuntu Server)

Primero se evita que cloud-init sobrescriba la red y el hostname:
```bash
printf "network: {config: disabled}\n" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
printf "preserve_hostname: true\n" | sudo tee /etc/cloud/cloud.cfg.d/99-preserve-hostname.cfg
```

Fichero `/etc/netplan/00-installer-config.yaml`:
```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp1s0:
      match:
        macaddress: 52:54:00:4e:49:be
      set-name: enp1s0
      dhcp4: false
      addresses:
        - 192.168.102.2/24
      routes:
        - to: default
          via: 192.168.102.254
      nameservers:
        addresses: [192.168.101.1]
  version: 2
```

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

### cliente1 (Fedora)

Fedora usa NetworkManager. El nombre de la conexión se ve con:
```bash
nmcli con show
```
```
NAME                 UUID                                  TYPE      DEVICE
Conexión cableada 1  c81fb4c6-7e17-352a-a408-660c461ea1be  ethernet  enp1s0
```

Se usa el UUID para evitar problemas con la tilde y los espacios:
```bash
sudo nmcli con mod c81fb4c6-7e17-352a-a408-660c461ea1be \
  ipv4.method manual \
  ipv4.addresses 172.20.0.2/16 \
  ipv4.gateway 172.20.0.1 \
  ipv4.dns 192.168.101.1 \
  ipv6.method disabled

sudo nmcli con up c81fb4c6-7e17-352a-a408-660c461ea1be
```

La configuración queda guardada en `/etc/NetworkManager/system-connections/Conexión cableada 1.nmconnection`.

Para aligerar la máquina se configuró el arranque en modo texto:
```bash
sudo systemctl set-default multi-user.target
```

### cliente2 (Windows 11)

Durante la instalación, Windows exige internet. Como br-red2 no tiene DHCP, se omitió creando una cuenta local: **Shift + F10** y
```
start ms-cxh:localonly
```

Configuración de red en PowerShell como administrador:
```powershell
Get-NetAdapter
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 172.20.0.3 -PrefixLength 16 -DefaultGateway 172.20.0.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.101.1
```

Windows bloquea el ping entrante por defecto, así que se permite:
```powershell
New-NetFirewallRule -DisplayName "Permitir ICMPv4" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

### Comprobación de conectividad

Las pruebas importantes son las que **cruzan el router** de una red a otra, porque demuestran que el forwarding funciona.

| Desde | Hacia | IP | Qué demuestra |
|---|---|---|---|
| router | web | 192.168.102.2 | br-red1 |
| router | cliente1 | 172.20.0.2 | br-red2 |
| router | cliente2 | 172.20.0.3 | br-red2 |
| web | cliente1 | 172.20.0.2 | **cruza el router** |
| web | cliente2 | 172.20.0.3 | **cruza el router** |
| cliente1 | cliente2 | 172.20.0.3 | misma red |
| cliente1 | web | 192.168.102.2 | **cruza el router** |
| cliente2 | web | 192.168.102.2 | **cruza el router** |

En Linux:
```bash
ping -c 3 192.168.102.2
```
En Windows:
```
ping 192.168.102.2
```

---

## 2. FQDN

### Router, web y cliente1

```bash
sudo hostnamectl set-hostname router.gabrielmerencio.org
```

Y en `/etc/hosts` la línea de `127.0.1.1`:
```
127.0.1.1   router.gabrielmerencio.org router
```

Lo mismo en las otras dos máquinas, cambiando el nombre:

| Máquina | Comando | Línea en `/etc/hosts` |
|---|---|---|
| web | `sudo hostnamectl set-hostname web.gabrielmerencio.org` | `127.0.1.1   web.gabrielmerencio.org web` |
| cliente1 | `sudo hostnamectl set-hostname cliente1.gabrielmerencio.org` | `127.0.1.1   cliente1.gabrielmerencio.org cliente1` |

Comprobación:
```bash
hostname -f
```
```
web.gabrielmerencio.org
```

### cliente2 (Windows)

Windows no admite puntos en el nombre del equipo, así que el FQDN se forma con el **nombre** y el **sufijo DNS principal**:
```powershell
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "NV Domain" -Value "gabrielmerencio.org"
Rename-Computer -NewName "cliente2" -Restart
```

Comprobación (`hostname` en Windows solo muestra el nombre corto):
```powershell
[System.Net.Dns]::GetHostEntry("").HostName
```
```
cliente2.gabrielmerencio.org
```

O con `ipconfig /all`:
```
Nombre de host. . . . . . . . . : cliente2
Sufijo DNS principal  . . . . . : gabrielmerencio.org
```

---

## 3. Usuario `gabriel` con sudo sin contraseña

Las máquinas ya tenían usuarios como `gabriel-router` o `gabriel-cliente1`, así que se renombraron a `gabriel` en todas.

### Renombrar el usuario

No se puede renombrar un usuario que tiene sesión abierta (ni siquiera si has hecho `sudo su` desde él), así que hay que entrar **directamente como root** en la consola de virt-manager.

En Ubuntu y Fedora root viene bloqueado, así que primero se le da contraseña:
```bash
sudo passwd root
```

Ya como root:
```bash
who
pkill -KILL -u gabriel-router
usermod -l gabriel gabriel-router
groupmod -n gabriel gabriel-router
usermod -d /home/gabriel -m gabriel
```

- `usermod -l` cambia el nombre del usuario.
- `groupmod -n` cambia el nombre de su grupo.
- `usermod -d ... -m` mueve la carpeta personal (con su `.ssh`).

En **Fedora**, además, hay que restaurar las etiquetas de SELinux de la carpeta movida; si no, SSH puede ignorar las claves:
```bash
restorecon -Rv /home/gabriel
```

Al terminar, se vuelve a bloquear root:
```bash
sudo passwd -l root
```

### sudo sin contraseña

Se usa un fichero propio en `/etc/sudoers.d/` en lugar de editar `/etc/sudoers`, para que no se pierda al actualizar el paquete `sudo`:
```bash
sudo visudo -f /etc/sudoers.d/gabriel
```
```
gabriel ALL=(ALL) NOPASSWD:ALL
```

En Fedora `visudo` abre `vi` por defecto; para usar nano: `sudo EDITOR=nano visudo -f /etc/sudoers.d/gabriel`.

El fichero debe tener permisos `0440`; si no, `visudo -c` da el error *bad permissions, should be mode 0440*:
```bash
sudo chmod 0440 /etc/sudoers.d/gabriel
sudo chown root:root /etc/sudoers.d/gabriel
sudo visudo -c
```

### Comprobación

`sudo -k` borra la contraseña que sudo tiene recordada, para que la prueba sea real:
```bash
sudo -k
sudo whoami
```
```
root
```

---

## 4. Acceso por SSH con clave pública

### Resolución de nombres en el portátil

`/etc/hosts` del portátil:
```
192.168.101.2    router.gabrielmerencio.org    router
192.168.102.2    web.gabrielmerencio.org       web
172.20.0.2       cliente1.gabrielmerencio.org  cliente1
172.20.0.3       cliente2.gabrielmerencio.org  cliente2
```

### Configuración de `~/.ssh/config`

Desde el portátil llego directamente al **router** (br-nat) y a **web** (br-red1 es aislada y el host tiene IP en ella). A **cliente1** no, porque br-red2 es muy aislada y el host no tiene IP ahí, así que hay que saltar por el router con `ProxyJump`.

```
Host router
    HostName 192.168.101.2

Host web
    HostName 192.168.102.2

Host cliente1
    HostName 172.20.0.2
    ProxyJump router

Host router web cliente1
    User gabriel
    IdentityFile ~/.ssh/id_rsa
```

```bash
chmod 600 ~/.ssh/config
```

En `HostName` se pone la **IP** y no el nombre: con un salto, el nombre del destino lo resuelve el router, no el portátil. Al principio usé `ssh -J gabriel-router@router gabriel-cliente1@cliente1` y fallaba con *Name or service not known* porque el router no conocía el nombre `cliente1`.

### Copiar la clave pública

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub router
ssh-copy-id -i ~/.ssh/id_rsa.pub web
ssh-copy-id -i ~/.ssh/id_rsa.pub cliente1
```

En Fedora puede hacer falta arrancar el servicio y abrir el firewall:
```bash
sudo systemctl enable --now sshd
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

### Comprobación

```bash
ssh router
ssh web
ssh cliente1
```

Entran sin pedir contraseña.

### ¿Para qué sirve `ssh -A`?

`ssh -A` activa el **reenvío del agente SSH**. Si entro al router con `ssh -A router` y desde ahí hago `ssh cliente1`, el router le pide al agente de **mi portátil** que haga la autenticación. La clave privada nunca sale del portátil.

**Problema de seguridad que evita:** si copiara mi `id_rsa` al router, la clave quedaría guardada allí. Si alguien entra en el router (o cualquiera con root), me la roba y puede acceder a todas mis máquinas cuando quiera, esté mi portátil encendido o no. Con `-A` no hay clave en el router que robar.

**Riesgo que sigue teniendo:** mientras estoy conectado, un root del router podría usar mi agente para autenticarse en mi nombre. Por eso solo se debe usar con máquinas de confianza. `ProxyJump` (`-J`) es todavía más seguro, porque ni siquiera expone el agente en el router; es lo que uso en el `~/.ssh/config`.

---

## 5. Acceso a internet de las máquinas internas

### El problema

El NAT de libvirt solo traduce la red `192.168.101.0/24`. Si web sale con su IP `192.168.102.2`, la respuesta no sabe volver, porque nadie fuera conoce esa red. El router tiene que hacer su propio NAT.

### Resolución DNS del router

Al intentar instalar paquetes en el router aparecía *Fallo temporal al resolver «deb.debian.org»*. Había dos problemas:

1. `/etc/resolv.conf` estaba vacío (solo comentarios). Lo había generado **dhcpcd** cuando la interfaz iba por DHCP, y al pasar a IP estática se quedó sin `nameserver`.
2. Con `1.1.1.1` u `8.8.8.8` seguía sin resolver, aunque `ping 1.1.1.1` funcionaba: **la red física del portátil bloquea el DNS hacia servidores externos**.

**Solución:** usar `192.168.101.1`. libvirt lanza un **dnsmasq** en las redes con IP (aunque no tengan DHCP), que recibe las consultas y las reenvía al DNS que usa el portátil, que sí está permitido. Además funciona en cualquier red donde esté el portátil.

```bash
printf "nameserver 192.168.101.1\n" | sudo tee /etc/resolv.conf
sudo systemctl disable --now dhcpcd
sudo chattr +i /etc/resolv.conf
```

- Se desactiva `dhcpcd` para que no vuelva a sobrescribir el fichero.
- `chattr +i` hace el fichero inmutable. Para editarlo en el futuro: `sudo chattr -i /etc/resolv.conf`.

Las máquinas internas usan también `192.168.101.1` como DNS; llegan a él a través del router.

### Regla de NAT con iptables

```bash
sudo apt install iptables iptables-persistent
```

Se usa **SNAT** con una regla por cada red interna, en modo lista blanca (solo sale lo autorizado):
```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.102.0/24 -o enp1s0 -j SNAT --to-source 192.168.101.2
sudo iptables -t nat -A POSTROUTING -s 172.20.0.0/16 -o enp1s0 -j SNAT --to-source 192.168.101.2
```

Persistencia (se guardan en `/etc/iptables/rules.v4` y se cargan en cada arranque):
```bash
sudo netfilter-persistent save
sudo systemctl enable netfilter-persistent
```

Comprobación:
```bash
sudo iptables -t nat -L POSTROUTING -n -v
```

### ¿Es necesario usar enmascaramiento?

Es necesario hacer **NAT de origen**, pero **no hace falta MASQUERADE**. MASQUERADE consulta la IP de la interfaz en cada paquete y está pensado para interfaces con **IP dinámica**. Como la IP del router en br-nat es **fija** (`192.168.101.2`), lo correcto es **SNAT**, que usa directamente esa IP y es más eficiente.

### Recorrido de un paquete

Cuando web hace `ping deb.debian.org`:

1. Pregunta al DNS `192.168.101.1`; el paquete va a su puerta de enlace, el router.
2. El router lo reenvía (forwarding) por br-nat y cambia el origen a `192.168.101.2` (SNAT).
3. El host hace su propio NAT hacia la red real y el paquete sale a internet.
4. La respuesta vuelve al router, que deshace el cambio y se la entrega a web.

### Comprobación

En web y cliente1:
```bash
ping -c 3 1.1.1.1
ping -c 3 deb.debian.org
```

En cliente2:
```
ping 1.1.1.1
ping deb.debian.org
```

- `ping 1.1.1.1` demuestra el **acceso a internet**.
- `ping deb.debian.org` demuestra la **resolución DNS** (antes de hacer ping tiene que traducir el nombre).

Después, en el router, los contadores `pkts` de las reglas SNAT han subido:
```bash
sudo iptables -t nat -L POSTROUTING -n -v
```

---

## 6. Servidor web y acceso desde el exterior

### ¿Se puede acceder a web desde el exterior sin pasar por el router?

**No.**

1. web tiene una IP privada (`192.168.102.2`) en una red aislada que solo existe dentro del portátil; desde fuera nadie tiene ruta hacia ella.
2. Su única tarjeta está en br-red1 y su puerta de enlace es el router: todo lo que entra o sale de web pasa por él.
3. El SNAT solo funciona hacia fuera: las respuestas vuelven porque la conexión la inició web, pero una conexión nueva desde fuera no tiene regla que la lleve a web.
4. Además hay dos niveles de NAT (el del portátil y el del router).

El portátil sí llega directamente a web, porque br-red1 es **aislada** y el host tiene IP en ella, pero el host forma parte del escenario, no del exterior. Para permitir el acceso desde fuera hace falta **DNAT** en el router.

### Instalar Apache en web

```bash
sudo apt update
sudo apt install apache2
printf "<h1>www.gabrielmerencio.org</h1>\n<p>Servidor web de Gabriel</p>\n" | sudo tee /var/www/html/index.html
systemctl is-active apache2
curl http://localhost
```

### Regla DNAT en el router

Todo lo que llega al puerto 80 por la interfaz exterior (`enp1s0`) se reenvía a web. Solo se abre el puerto 80 y solo en esa interfaz:
```bash
sudo iptables -t nat -A PREROUTING -i enp1s0 -p tcp --dport 80 -j DNAT --to-destination 192.168.102.2:80
sudo netfilter-persistent save
sudo iptables -t nat -L PREROUTING -n -v
```

La respuesta de web vuelve sola por el router, porque es su puerta de enlace, y el router deshace la traducción.

### Resolución estática

**Desde el exterior (portátil)**, apuntando a la IP del **router**, como pide el enunciado:
```
192.168.101.2    www.gabrielmerencio.org
```

**Desde la red muy aislada**, apuntando directamente a **web**, porque los clientes ya llegan a ella enrutando a través del router (sin NAT):

cliente1, en `/etc/hosts`:
```
192.168.102.2    www.gabrielmerencio.org
```

cliente2, en `C:\Windows\System32\drivers\etc\hosts` (PowerShell como administrador):
```powershell
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "192.168.102.2    www.gabrielmerencio.org"
```

### Comprobación

Portátil:
```bash
getent hosts www.gabrielmerencio.org
curl http://www.gabrielmerencio.org
```
Y en el navegador: `http://www.gabrielmerencio.org`

cliente1:
```bash
curl http://www.gabrielmerencio.org
```

cliente2:
```
curl http://www.gabrielmerencio.org
```
O en Edge: `http://www.gabrielmerencio.org`

En el router, el contador de la regla DNAT sube cada vez que el portátil carga la página (desde los clientes no sube, porque no pasan por el DNAT):
```bash
sudo iptables -t nat -L PREROUTING -n -v
```

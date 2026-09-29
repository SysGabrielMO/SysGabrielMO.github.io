---
title: "Router con SNAT, DNAT y DHCP — Parte 2: servidor DHCP con Kea"
asignatura: Servicios
date: 2026-09-29
resumen: "Sustituyo el direccionamiento estático de la parte 1 por DHCP con Kea en el router: creo los ámbitos de las dos redes internas, una reserva por MAC para el servidor web y capturo el proceso DORA con tcpdump. Compruebo qué les pasa a los clientes cuando el servidor se apaga o cambia de rango, y paso la interfaz pública del router a DHCP, cambiando SNAT por MASQUERADE para que el NAT siga funcionando."
tags:
  - DHCP
  - Kea
  - MASQUERADE
  - tcpdump
  - Netplan
---

> Módulo: SRI (Servicios de Red e Internet) · 2º ASIR
> Continuación de la parte 1, donde todo el escenario funcionaba con direccionamiento estático.

En esta parte se sustituye la configuración estática por **DHCP**, usando **Kea DHCP** en el router. Además de configurarlo, se estudia qué pasa cuando el servidor se cae o cambia su configuración con clientes activos, y cómo afecta al router que su interfaz pública pase a tomar IP por DHCP.

---

## 0. Escenario

```
            br-nat / default (NAT)
                    │
               ┌────┴────┐
               │ router  │  Debian · Kea DHCP · iptables
               └┬───────┬┘
     br-red1    │       │    br-red2
    (aislada)   │       │  (muy aislada)
         ┌──────┘       └──────┬──────────┐
       ┌─┴──┐            ┌─────┴───┐ ┌────┴────┐
       │ web│            │cliente1 │ │cliente2 │
       └────┘            │ Fedora  │ │Windows11│
     Ubuntu Server       └─────────┘ └─────────┘
```

| Red | Bridge | Tipo | Direccionamiento |
|---|---|---|---|
| br-nat | br-nat | NAT sin DHCP | 192.168.101.0/24 (host .1) |
| default | virbr0 | NAT con DHCP | 192.168.122.0/24 (host .1) — se usa desde la tarea 9 |
| br-red1 | br-red1 | Aislada | 192.168.102.0/24 (host .1) |
| br-red2 | br-red2 | Muy aislada | 172.20.0.0/16 (host sin IP) |

| Máquina | Interfaz | Red | IP |
|---|---|---|---|
| router | enp1s0 | br-nat → default | 192.168.101.2 → DHCP (192.168.122.205) |
| router | enp2s0 | br-red1 | 192.168.102.254/24 |
| router | enp3s0 | br-red2 | 172.20.0.1/16 |
| web | enp1s0 | br-red1 | 192.168.102.2 (reserva DHCP) |
| cliente1 | enp1s0 | br-red2 | DHCP |
| cliente2 | Ethernet | br-red2 | DHCP |

---

## 1. Conceptos previos de DHCP

### El proceso DORA

Cuando un cliente sin IP pide configuración, se intercambian cuatro mensajes:

| Mensaje | Origen → destino | Significado |
|---|---|---|
| **DISCOVER** | 0.0.0.0 → 255.255.255.255 | El cliente, sin IP, pregunta a toda la red si hay un servidor DHCP. |
| **OFFER** | servidor → IP ofrecida | El servidor le ofrece una IP y sus opciones. |
| **REQUEST** | 0.0.0.0 → 255.255.255.255 | El cliente acepta la oferta. Va en broadcast para que otros servidores sepan que no ha elegido la suya. |
| **ACK** | servidor → IP asignada | El servidor confirma. Desde aquí el cliente usa la IP. |

Los puertos son **UDP 67** (servidor) y **UDP 68** (cliente).

### La concesión (lease) y sus tiempos

La IP no se regala, se **presta** durante un tiempo:

| Parámetro | Por defecto | Qué hace el cliente |
|---|---|---|
| **T1** (`renew-timer`) | 50 % de la concesión | Pide renovar a **su** servidor, en **unicast**. |
| **T2** (`rebind-timer`) | 87,5 % | Si su servidor no respondió, pide renovar a **cualquier** servidor, en **broadcast**. |
| **Caducidad** (`valid-lifetime`) | 100 % | Debe dejar de usar la IP y empezar de cero con DISCOVER. |

Siempre debe cumplirse **T1 < T2 < valid-lifetime**.

> 💡 Idea clave para toda la práctica: **en DHCP siempre inicia el cliente**. El servidor nunca avisa de nada; solo responde cuando el cliente pregunta.

---

## 2. Tarea 1 — Instalar Kea y crear el ámbito de la red muy aislada

### Plan de direcciones

| Dato | Valor | Motivo |
|---|---|---|
| Red / máscara | 172.20.0.0/16 → 255.255.0.0 | Red de br-red2 |
| Rango | 172.20.0.100 – 172.20.0.200 | Deja libres las IPs bajas (router en .1) |
| Puerta de enlace | 172.20.0.1 | El router en br-red2 |
| DNS | 192.168.101.1 | dnsmasq de libvirt en br-nat (ver nota) |
| Broadcast | 172.20.255.255 | Última dirección de un /16 |
| Concesión | 1800 s | 30 minutos |

> **¿Por qué un DNS que no está en la red del cliente?** La puerta de enlace sí tiene que estar en la misma red (es el primer salto), pero el DNS es solo una IP a la que mandar preguntas: el cliente la alcanza a través de su puerta de enlace. En este escenario se usa el dnsmasq de libvirt porque la red del instituto bloquea las consultas a DNS externos (1.1.1.1, 8.8.8.8).

### Instalación

```bash
sudo apt update
sudo apt install kea-dhcp4-server
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.orig
```

La copia `.orig` guarda el fichero de ejemplo original: sirve para volver atrás si algo se rompe y como chuleta de opciones.

### Estructura del fichero `/etc/kea/kea-dhcp4.conf`

El fichero es **JSON** (Kea admite además comentarios con `//` y `#`). Los bloques que se usan:

| Bloque | Para qué sirve |
|---|---|
| `interfaces-config` | En qué interfaces escucha Kea. **Nunca** poner la interfaz pública: respondería a la red del host. |
| `lease-database` | Dónde guarda las concesiones. `memfile` = fichero CSV en `/var/lib/kea/kea-leases4.csv`. |
| `renew-timer`, `rebind-timer`, `valid-lifetime` | T1, T2 y duración de la concesión (en segundos). |
| `option-data` (global) | Opciones comunes a **todas** las subredes (DNS, dominio). Las subredes las heredan. |
| `subnet4` | Lista de subredes (ámbitos). Cada una con su `id`, rango (`pools`) y opciones propias. |

Del fichero de ejemplo se eliminan todas las opciones y bloques de demostración (`domain-search`, `boot-file-name`, `client-classes`, reservas de ejemplo con IPs `192.0.2.x`…), porque si no se enviarían datos basura a los clientes o Kea daría error al no pertenecer esas IPs a nuestra subred.

### Configuración de la tarea 1

```json
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": [ "enp3s0" ],
        "service-sockets-max-retries": 10,
        "service-sockets-retry-wait-time": 5000
    },

    "lease-database": {
        "type": "memfile",
        "lfc-interval": 3600
    },

    "renew-timer": 900,
    "rebind-timer": 1600,
    "valid-lifetime": 1800,

    "option-data": [
        {
            "name": "domain-name-servers",
            "data": "192.168.101.1"
        },
        {
            "code": 15,
            "data": "gabrielmerencio.org"
        }
    ],

    "subnet4": [
        {
            "id": 1,
            "interface": "enp3s0",
            "subnet": "172.20.0.0/16",
            "pools": [ { "pool": "172.20.0.100 - 172.20.0.200" } ],
            "option-data": [
                {
                    "name": "routers",
                    "data": "172.20.0.1"
                },
                {
                    "name": "broadcast-address",
                    "data": "172.20.255.255"
                }
            ]
        }
    ]
}
}
```

Dónde está cada requisito del enunciado:

| Requisito | Dónde |
|---|---|
| Rango | `pools` |
| Máscara | Kea la deduce del `/16` de `subnet` y la envía sola (opción 1) |
| Puerta de enlace | `routers` |
| DNS | `domain-name-servers` (global) |
| Broadcast | `broadcast-address` |
| 30 minutos | `valid-lifetime: 1800` |

`"code": 15` es la opción **domain-name** (equivale a `"name": "domain-name"`).

### Arrancar y comprobar

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl enable kea-dhcp4-server
systemctl status kea-dhcp4-server
sudo ss -ulpn | grep 67
```

`ss` debe mostrar `kea-dhcp4` escuchando en el puerto 67.

Para ver en directo lo que hace el servidor (muy útil en todas las pruebas):

```bash
sudo journalctl -fu kea-dhcp4-server
```

### ⚠️ Problemas que pueden aparecer

| Síntoma | Causa | Solución |
|---|---|---|
| `Invalid character: ;` con línea y columna | Un `;` en lugar de `,` | Ir a la línea con `nano -l` (números) y `Ctrl+_` |
| El servicio no arranca, sin más | Coma de más o de menos en el JSON | El **último** elemento de una lista/bloque no lleva coma. Ver el error con `journalctl -u kea-dhcp4-server -n 20 --no-pager` (el `--no-pager` evita que las líneas salgan cortadas) |
| `kea-dhcp4 -t` dice `Unable to open file` | Restricción al lanzarlo a mano | Validar arrancando el servicio y mirando el log: indica línea y columna igual |
| Aviso `ConfigurationDirectory 'kea' ... mode is different` | Permisos de `/etc/kea` (750 vs 755) | Solo es un aviso de systemd, se puede ignorar |
| Tras reiniciar el router, **nadie recibe IP** aunque Kea está `active` | Kea arrancó antes de que `enp3s0` tuviera IP, no pudo abrir el socket y no lo reintenta | Añadir `service-sockets-max-retries` y `service-sockets-retry-wait-time` en `interfaces-config` (ya incluido arriba). Arreglo rápido: `systemctl restart kea-dhcp4-server` |

---

## 3. Tarea 2 — Clientes con configuración dinámica

### cliente1 (Fedora, NetworkManager)

> Hacerlo desde la **consola de virt-manager**, no por SSH: al cambiar la IP se corta la sesión.

Ver el nombre de la conexión y su estado actual:

```bash
nmcli con show
nmcli con show "Conexión cableada 1" | grep ipv4
```

Antes: `ipv4.method: manual`, con IP, puerta de enlace y DNS a mano. Se pasa a DHCP:

```bash
sudo nmcli con mod "Conexión cableada 1" ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""
sudo nmcli con up "Conexión cableada 1"
```

- `ipv4.method auto` → DHCP.
- Vaciar `addresses`, `gateway` y `dns` es imprescindible: si no, se **mezclan** los datos manuales con los del DHCP.
- `nmcli con up` reactiva la conexión para aplicar los cambios.

Comprobación:

```bash
nmcli con show "Conexión cableada 1" | grep ipv4.method   # auto
ip -br a show enp1s0
ip r
resolvectl status enp1s0
nmcli -f DHCP4 con show "Conexión cableada 1"
```

El último comando muestra **todo lo que envió el servidor**: `ip_address`, `subnet_mask = 255.255.0.0`, `routers = 172.20.0.1`, `broadcast_address`, `domain_name_servers`, `domain_name`, `dhcp_lease_time = 1800`, `dhcp_server_identifier = 172.20.0.1` y `expiry` (caducidad en formato epoch).

### cliente2 (Windows 11)

**Opción gráfica** (la más sencilla):

- `Win + R` → `ncpa.cpl` → clic derecho en **Ethernet** → **Propiedades** → **Protocolo de Internet versión 4** → **Propiedades** → marcar *Obtener una dirección IP automáticamente* y *Obtener la dirección del servidor DNS automáticamente*.

**Opción PowerShell** (como administrador):

```powershell
Remove-NetRoute -InterfaceAlias Ethernet -DestinationPrefix 0.0.0.0/0 -Confirm:$false
Remove-NetIPAddress -InterfaceAlias Ethernet -AddressFamily IPv4 -Confirm:$false
Set-NetIPInterface -InterfaceAlias Ethernet -Dhcp Enabled
Set-DnsClientServerAddress -InterfaceAlias Ethernet -ResetServerAddresses
```

Aplicar y comprobar:

```powershell
ipconfig /renew
ipconfig /all
```

En `ipconfig /all` deben aparecer: *DHCP habilitado: sí*, la IP, máscara 255.255.0.0, puerta de enlace 172.20.0.1, servidor DHCP 172.20.0.1, DNS, sufijo `gabrielmerencio.org` y las fechas de *concesión obtenida* y *expira* (30 min de diferencia).

> Si `ipconfig /renew` se queda esperando mucho, **nadie responde al DHCP**: casi siempre es que Kea no está escuchando (ver problema del arranque en la tarea 1).

### Lista de concesiones en el servidor

```bash
sudo column -s, -t /var/lib/kea/kea-leases4.csv
```

Columnas importantes: `address`, `hwaddr` (MAC), `valid_lifetime`, `expire` (epoch), `hostname` y `state`.

> **Cómo leer el fichero:** Kea **va añadiendo una línea** por cada cambio de una concesión (asignación, renovación, liberación). Por eso una misma IP aparece varias veces. La válida es **la última** de cada IP con `state 0` (activa); `state 2` son concesiones ya caducadas/recuperadas. Periódicamente (`lfc-interval`) Kea compacta el fichero.

### Conectividad al exterior

```bash
ping -c 3 1.1.1.1          # salida a internet (puerta de enlace + NAT)
ping -c 3 deb.debian.org   # resolución DNS
```

En Windows igual, sin `-c 3`.

---

## 4. Tarea 3 — Capturar el DORA con tcpdump

En el router:

```bash
sudo apt install tcpdump
sudo tcpdump -i enp3s0 -n -v 'udp port 67 or udp port 68'
```

| Opción | Qué hace |
|---|---|
| `-i enp3s0` | Captura en la interfaz de br-red2 |
| `-n` | No traduce IPs a nombres |
| `-v` | Muestra el detalle, incluido el tipo de mensaje DHCP |
| `'udp port 67 or udp port 68'` | Solo tráfico DHCP |

Para forzar una concesión completa, desde **Windows**:

```powershell
ipconfig /release
ipconfig /renew
```

En cada paquete se busca la línea `DHCP-Message (53)`: aparecen en orden **Discover → Offer → Request → ACK** (antes, un Release por el `/release`). En el Offer y el ACK se ven todas las opciones configuradas: `Subnet-Mask`, `Default-Gateway`, `Domain-Name-Server`, `BR` (broadcast), `Lease-Time`, `RN`/`RB` (T1/T2)…

> **¿Por qué con Windows y no con Fedora?** NetworkManager recuerda su última IP y la pide directamente con un REQUEST (estado **INIT-REBOOT**), saltándose DISCOVER y OFFER. Tras un `/release`, Windows ya no tiene IP y hace el proceso completo.

> **Detalle curioso:** Kea solo envía las opciones que el cliente **pide** en su `Parameter-Request (55)`. Fedora pide `BR (28)` y recibe el broadcast; Windows no lo pide y no lo recibe (lo calcula él a partir de IP y máscara).

---

## 5. Tarea 4 — ¿Qué pasa si el servidor DHCP se apaga?

### Preparación

Para no esperar 30 minutos, se baja la concesión a 2 minutos:

```json
"renew-timer": 60,
"rebind-timer": 105,
"valid-lifetime": 120,
```

```bash
sudo systemctl restart kea-dhcp4-server
```

Los clientes siguen con su concesión de 30 min hasta que pidan otra, así que se fuerza:

- cliente1: `sudo nmcli con up "Conexión cableada 1"`
- cliente2: `ipconfig /renew`

Comprobación: `nmcli -f DHCP4 con show "Conexión cableada 1" | grep lease_time` → `120`; en Windows, 2 minutos entre *obtenida* y *expira*.

Se dejan observando: el `tcpdump` en el router, `watch -n 5 ip -br a show enp1s0` en cliente1 e `ipconfig` en cliente2. Y se apaga el servidor **justo después** de que los clientes cojan la concesión:

```bash
sudo systemctl stop kea-dhcp4-server
```

### Resultado (captura real)

cliente1 (`52:54:00:0e:26:99`), concesión obtenida a las 18:53:57:

| Hora | Paquete | Qué ocurre |
|---|---|---|
| 18:54:57 | `172.20.0.100 > 172.20.0.1` Request | **T1 (+60 s)**: renovación en unicast a su servidor. Sigue usando su IP (`Client-IP 172.20.0.100`). Sin respuesta. |
| 18:55:42 | `0.0.0.0 > 255.255.255.255` Request | **T2 (+105 s)**: renovación en broadcast a cualquier servidor. Sin respuesta. |
| 18:55:57 | `0.0.0.0 > 255.255.255.255` Discover | **Caducada (+120 s)**: ya no tiene IP, empieza de cero pidiendo la misma (`Requested-IP 172.20.0.100`). |

cliente2 (`52:54:00:84:a8:22`) empieza a mandar Discover al caducar su concesión, también pidiendo su IP anterior (`Requested-IP 172.20.0.101`).

Estado de los clientes:

| Momento | Linux (cliente1) | Windows (cliente2) |
|---|---|---|
| Concesión vigente | IP y `ping` funcionando | IP y `ping` funcionando |
| T1 y T2 | Intenta renovar, nadie responde | Intenta renovar, nadie responde |
| Caducada | **Sin IPv4**, `ping` falla | IP **APIPA 169.254.x.x**, `ping` falla |
| Servidor arrancado de nuevo | Recupera la IP (reintenta cada pocos segundos) | Recupera la IP (reintenta más espaciado; se puede forzar con `ipconfig /renew`) |

### Por qué ocurre

1. **Mientras la concesión es válida**, el cliente puede usar la IP aunque el servidor no exista: no lo necesita para nada.
2. En **T1** intenta renovar con su servidor y en **T2** con cualquiera. Nadie responde.
3. Al **caducar**, el cliente está **obligado** a dejar la IP (RFC 2131), porque el servidor podría habérsela dado a otro equipo.
   - **Linux** se queda sin dirección IPv4 y sigue mandando DISCOVER.
   - **Windows** se asigna una dirección **APIPA** (169.254.0.0/16), que solo sirve para hablar con equipos del mismo enlace: sin puerta de enlace, no sale al exterior.
4. Como los clientes siguen buscando servidor, en cuanto vuelve se recuperan solos.

---

## 6. Tarea 5 — ¿Qué pasa si se cambia la configuración con concesiones activas?

### Procedimiento

Truco para hacerlo a tiempo: **editar el fichero antes, pero reiniciar Kea después** de que los clientes cojan concesión (Kea sigue con la configuración antigua hasta que se reinicia).

1. Con los tiempos aún en 60 / 105 / 120, cambiar el pool **sin reiniciar**:
   ```json
   "pools": [ { "pool": "172.20.0.50 - 172.20.0.99" } ],
   ```
2. Los clientes cogen concesión (del rango viejo): `nmcli con up` / `ipconfig /renew` → cliente1 = .100, cliente2 = .101.
3. Enseguida: `sudo systemctl restart kea-dhcp4-server`.
4. Observar con `tcpdump` y `journalctl -fu kea-dhcp4-server`.

### Resultado (log real de Kea)

```
11:20:34  DHCPREQUEST received from 172.20.0.100 to 172.20.0.1       ← cliente1, T1 → sin respuesta
11:20:38  DHCPREQUEST received from 172.20.0.101 to 172.20.0.1       ← cliente2, renovación → sin respuesta
11:21:19  DHCPREQUEST received from 0.0.0.0 to 255.255.255.255       ← cliente1, T2 → sin respuesta
11:21:35  DHCPDISCOVER received from 0.0.0.0                         ← cliente1, caducada
11:21:35  lease 172.20.0.50 has been allocated for 120 seconds       ← IP del rango NUEVO
11:21:38  client is in INIT-REBOOT state and requests address 172.20.0.101  ← Windows insiste → sin respuesta
11:21:44  DHCPDISCOVER received from 0.0.0.0                         ← cliente2 empieza de cero
11:21:58  lease 172.20.0.53 has been allocated for 120 seconds       ← IP del rango NUEVO
```

Y la comparación que lo demuestra todo — una renovación de la IP **nueva**:

```
11:22:35  172.20.0.50 > 172.20.0.1   Request  (T1 de la concesión nueva)
11:22:35  172.20.0.1  > 172.20.0.50  ACK      ← respuesta inmediata
```

### Por qué ocurre

- **Mientras dura la concesión** no cambia nada: el servidor no puede avisar del cambio porque en DHCP siempre inicia el cliente.
- **Al renovar**, el cliente pide su IP antigua, que ya no pertenece a ningún rango. Kea funciona por defecto en modo **no autoritativo** (`"authoritative": false`): no rechaza con NAK las direcciones que no puede conceder, **simplemente ignora la petición**. Para el cliente es como si el servidor no existiera.
- **Al caducar**, el cliente empieza de cero con DISCOVER y recibe una IP del rango nuevo. Windows, antes, intenta una vez más recuperar su IP en estado **INIT-REBOOT**.
- La misma petición de renovación, con una IP que sí está en el rango, recibe **ACK al instante**: el servidor valida cada petición con su **configuración actual**.

> **Para investigar:** con `"authoritative": true` dentro de `Dhcp4`, Kea sí responde **DHCPNAK** a las renovaciones de IPs que ya no puede dar, y los clientes cambian de IP en T1 sin esperar a que caduque.

> **¿Por qué Windows acabó con la .53 y no la .51?** Envió varios DISCOVER antes de aceptar una oferta. Kea no reserva la IP ofrecida, así que a cada DISCOVER le ofreció la siguiente libre.

### Al terminar

Volver a dejar el pool en `172.20.0.100 - 172.20.0.200` y los tiempos en `900 / 1600 / 1800`, reiniciar Kea y forzar una concesión nueva en los clientes (en Windows `ipconfig /release` + `ipconfig /renew`, para que no insista con su IP antigua).

---

## 7. Tarea 6 — Nuevo ámbito para la red aislada (24 horas)

| Dato | Valor | Motivo |
|---|---|---|
| Red / máscara | 192.168.102.0/24 → 255.255.255.0 | Red de br-red1 |
| Rango | 192.168.102.100 – 192.168.102.200 | Excluye .1 (host), .2 (web) y .254 (router) |
| Puerta de enlace | 192.168.102.254 | El router en br-red1 |
| DNS | heredado de las opciones globales | |
| Broadcast | 192.168.102.255 | Última dirección de un /24 |
| Concesión | 86400 s | 24 horas |

Cambios en el fichero:

1. Añadir `enp2s0` a `interfaces`: `"interfaces": [ "enp3s0", "enp2s0" ]`.
2. Añadir una **segunda subred** dentro de `subnet4`, separada de la primera por una **coma** (`},`):

```json
{
    "id": 2,
    "interface": "enp2s0",
    "subnet": "192.168.102.0/24",
    "renew-timer": 43200,
    "rebind-timer": 75600,
    "valid-lifetime": 86400,
    "pools": [ { "pool": "192.168.102.100 - 192.168.102.200" } ],
    "option-data": [
        { "name": "routers", "data": "192.168.102.254" },
        { "name": "broadcast-address", "data": "192.168.102.255" }
    ]
}
```

> **Herencia en Kea:** lo que está a nivel global (tiempos, opciones) lo heredan todas las subredes, salvo que la subred defina su propio valor. Por eso esta subred lleva sus **propios tiempos**: si heredase los globales, web tendría concesiones de 30 min. Los T1/T2 elegidos son el 50 % y el 87,5 % de 24 h. El DNS y el dominio, en cambio, se heredan sin tocar nada.

---

## 8. Tarea 7 — Reserva para el servidor web

Una **reserva** hace que un equipo concreto reciba siempre la misma IP por DHCP. Kea lo identifica por su **MAC**:

```bash
ip link show enp1s0        # en web → link/ether 52:54:00:4e:49:be
```

Se añade en la **subred 2**, detrás de `option-data` (cuyo `]` pasa a ser `],`):

```json
"reservations": [
    {
        "hw-address": "52:54:00:4e:49:be",
        "ip-address": "192.168.102.2",
        "hostname": "web"
    }
]
```

> La IP reservada (.2) está **fuera del pool** a propósito: así Kea nunca se la dará a otro equipo.

---

## 9. Tarea 8 — Servidor web por DHCP

> Hacerlo desde la consola de virt-manager: al aplicar netplan la interfaz se reinicia.

`/etc/netplan/50-cloud-init.yaml` (o el fichero que haya en `/etc/netplan/`) pasa de IP fija a:

```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: true
      dhcp-identifier: mac
```

- Se eliminan `addresses`, `routes` y `nameservers`: todo llega por DHCP.
- **`dhcp-identifier: mac`** es importante: por defecto Ubuntu (systemd-networkd) se identifica ante el servidor con un ID propio, no con la MAC.
- En YAML la sangría es con **espacios**, nunca tabuladores.

```bash
sudo netplan apply
```

En el log de Kea: `lease 192.168.102.2 has been allocated for 86400 seconds`.

Comprobación:

```bash
ip -br a show enp1s0        # 192.168.102.2/24
ip r                         # default via 192.168.102.254 proto dhcp
resolvectl status enp1s0
networkctl status enp1s0    # Address ... (DHCPv4 via 192.168.102.254), DHCPv4 Client ID = MAC
```

> En `ip r` aparecen dos rutas extra (`192.168.101.1 via 192.168.102.254` y `192.168.102.254 dev enp1s0`). Son normales: systemd-networkd añade rutas hacia el DNS y la puerta de enlace que recibe por DHCP.

### Acceso a la web

Como web conserva su IP gracias a la reserva, el DNAT del router y los `/etc/hosts` siguen valiendo sin tocar nada:

```bash
curl http://www.gabrielmerencio.org     # desde el portátil (exterior, vía DNAT) y desde cliente1
```

```powershell
curl.exe http://www.gabrielmerencio.org  # desde cliente2 (o con el navegador)
```

---

## 10. Tarea 9 — La interfaz pública del router por DHCP

Se cambia la tarjeta de `br-nat` (sin DHCP) a la red **`default`** de libvirt (NAT con DHCP, 192.168.122.0/24). Cambiando la **misma** tarjeta, se conserva la MAC y el nombre `enp1s0`, así que las reglas de iptables siguen valiendo.

### En el host

```bash
sudo virsh net-list --all       # default: activo y con inicio automático
```

Si no lo está: `sudo virsh net-start default` y `sudo virsh net-autostart default`.

Con el router **apagado**: virt-manager → router → Detalles → NIC de br-nat → **Fuente de red: Red virtual 'default'** → Aplicar. Comprobar que la MAC no cambia.

### En el router (por consola)

`/etc/network/interfaces`:

```
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto enp1s0
iface enp1s0 inet dhcp

auto enp2s0
iface enp2s0 inet static
    address 192.168.102.254/24

auto enp3s0
iface enp3s0 inet static
    address 172.20.0.1/16
```

- `inet dhcp`: IP, puerta de enlace y DNS llegan por DHCP.
- Se quita el `gateway` estático. Como las interfaces internas no llevan `gateway`, **solo habrá una ruta por defecto**, la de `enp1s0`.
- ⚠️ No borrar los bloques de `enp2s0` y `enp3s0`: aunque sigan funcionando hasta el siguiente reinicio, sin ellos el router perdería las redes internas.

En la parte 1 se hizo `/etc/resolv.conf` inmutable; hay que desbloquearlo para que el DHCP pueda escribir el DNS:

```bash
sudo chattr -i /etc/resolv.conf
sudo reboot
```

Comprobación:

```bash
ip -br a show enp1s0     # 192.168.122.205/24
ip r                     # default via 192.168.122.1 dev enp1s0 proto dhcp
cat /etc/resolv.conf     # nameserver 192.168.122.1
```

```
default via 192.168.122.1 dev enp1s0 proto dhcp src 192.168.122.205 metric 1002
172.20.0.0/16 dev enp3s0 proto kernel scope link src 172.20.0.1
192.168.102.0/24 dev enp2s0 proto kernel scope link src 192.168.102.254
192.168.122.0/24 dev enp1s0 proto dhcp scope link src 192.168.122.205 metric 1002
```

Si `resolv.conf` sigue con el DNS antiguo, se pone a mano:

```bash
printf "nameserver 192.168.122.1\n" | sudo tee /etc/resolv.conf
```

### ⚠️ Consecuencias del cambio

| Qué se rompe | Por qué | Solución |
|---|---|---|
| Los equipos internos pierden internet | Las reglas SNAT ponen como origen 192.168.101.2, IP que el router ya no tiene | Tarea 10 (MASQUERADE) |
| Los equipos internos no resuelven nombres | Kea reparte el DNS 192.168.101.1, que ya no es alcanzable | Cambiar `domain-name-servers` a `192.168.122.1` en Kea |
| Acceso a la web y SSH desde el portátil | Apuntan a 192.168.101.2 | Actualizar `/etc/hosts` y `~/.ssh/config` del portátil con la IP nueva |

> Opcional: para que la IP pública del router no cambie nunca, se puede crear una reserva en la red `default`:
> `sudo virsh net-update default add ip-dhcp-host "<host mac='MAC' ip='192.168.122.2'/>" --live --config`

---

## 11. Tarea 10 — SNAT → MASQUERADE

| | SNAT | MASQUERADE |
|---|---|---|
| IP de origen | Fija, escrita en la regla (`--to-source`) | La que tenga la interfaz de salida **en cada momento** |
| Uso | Interfaz pública con IP **estática** | Interfaz pública con IP **dinámica** (DHCP) |
| Si la IP cambia | La regla deja de funcionar | Sigue funcionando; además olvida las conexiones traducidas si la interfaz cae |

Cambio de reglas (se mantiene una por red interna, en modo lista blanca):

```bash
sudo iptables -t nat -D POSTROUTING -s 192.168.102.0/24 -o enp1s0 -j SNAT --to-source 192.168.101.2
sudo iptables -t nat -D POSTROUTING -s 172.20.0.0/16 -o enp1s0 -j SNAT --to-source 192.168.101.2
sudo iptables -t nat -A POSTROUTING -s 192.168.102.0/24 -o enp1s0 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s 172.20.0.0/16 -o enp1s0 -j MASQUERADE
```

La regla **DNAT** del puerto 80 no se toca: solo depende de la interfaz (`-i enp1s0`), no de la IP.

Persistencia y comprobación tras reiniciar:

```bash
sudo netfilter-persistent save
cat /etc/iptables/rules.v4
sudo reboot
sudo iptables -t nat -L -n -v
systemctl status netfilter-persistent
```

`rules.v4` resultante:

```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A PREROUTING -i enp1s0 -p tcp -m tcp --dport 80 -j DNAT --to-destination 192.168.102.2:80
-A POSTROUTING -s 192.168.102.0/24 -o enp1s0 -j MASQUERADE
-A POSTROUTING -s 172.20.0.0/16 -o enp1s0 -j MASQUERADE
COMMIT
```

---

## 12. Comprobación final

### Actualizar el DNS de los clientes

Tras cambiar el DNS en Kea a `192.168.122.1` y reiniciarlo, los clientes lo reciben **al renovar** (lo visto en la tarea 5), así que se fuerza:

| Equipo | Comando |
|---|---|
| cliente1 | `sudo nmcli con up "Conexión cableada 1"` |
| cliente2 | `ipconfig /renew` |
| web | `sudo networkctl renew enp1s0` (su concesión es de 24 h) |

> Si en web sigue sin resolver, revisar `resolvectl status`: puede quedar el DNS antiguo guardado. Soluciones según el caso: `sudo networkctl reconfigure enp1s0`; si `/etc/resolv.conf` no es un enlace a `/run/systemd/resolve/stub-resolv.conf`, restaurarlo con `ln -sf`; o quitar un `DNS=` global de `/etc/systemd/resolved.conf`.

### Conectividad

En web, cliente1 y cliente2:

```bash
ping -c 3 1.1.1.1
ping -c 3 deb.debian.org
```

En el router, los contadores de las dos reglas MASQUERADE deben subir:

```bash
sudo iptables -t nat -L POSTROUTING -n -v
```

```
 pkts bytes target     prot opt in     out     source               destination
   62  4486 MASQUERADE  all  --  *      enp1s0  192.168.102.0/24     0.0.0.0/0
  262 17273 MASQUERADE  all  --  *      enp1s0  172.20.0.0/16        0.0.0.0/0
```

---

## 13. Fichero final de Kea

```json
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": [ "enp3s0", "enp2s0" ],
        "service-sockets-max-retries": 10,
        "service-sockets-retry-wait-time": 5000
    },

    "lease-database": {
        "type": "memfile",
        "lfc-interval": 3600
    },

    "renew-timer": 900,
    "rebind-timer": 1600,
    "valid-lifetime": 1800,

    "option-data": [
        {
            "name": "domain-name-servers",
            "data": "192.168.122.1"
        },
        {
            "code": 15,
            "data": "gabrielmerencio.org"
        }
    ],

    "subnet4": [
        {
            "id": 1,
            "interface": "enp3s0",
            "subnet": "172.20.0.0/16",
            "pools": [ { "pool": "172.20.0.100 - 172.20.0.200" } ],
            "option-data": [
                { "name": "routers", "data": "172.20.0.1" },
                { "name": "broadcast-address", "data": "172.20.255.255" }
            ]
        },
        {
            "id": 2,
            "interface": "enp2s0",
            "subnet": "192.168.102.0/24",
            "renew-timer": 43200,
            "rebind-timer": 75600,
            "valid-lifetime": 86400,
            "pools": [ { "pool": "192.168.102.100 - 192.168.102.200" } ],
            "option-data": [
                { "name": "routers", "data": "192.168.102.254" },
                { "name": "broadcast-address", "data": "192.168.102.255" }
            ],
            "reservations": [
                {
                    "hw-address": "52:54:00:4e:49:be",
                    "ip-address": "192.168.102.2",
                    "hostname": "web"
                }
            ]
        }
    ]
}
}
```

---

## 14. Resumen para estudiar

- **DORA**: Discover → Offer → Request → ACK. Puertos UDP 67 (servidor) y 68 (cliente).
- **La concesión es un préstamo**: T1 (50 %) renueva con su servidor en unicast, T2 (87,5 %) con cualquiera en broadcast, al caducar debe soltar la IP.
- **El servidor nunca inicia la comunicación**: ni su caída ni sus cambios afectan al cliente hasta que este renueva o caduca.
- **Sin servidor**: Linux se queda sin IP; Windows usa **APIPA** (169.254.0.0/16).
- **Kea no autoritativo** (por defecto) ignora las peticiones de IPs que no puede dar; autoritativo responde **NAK**.
- **Herencia**: opciones y tiempos globales se aplican a todas las subredes salvo que la subred defina los suyos.
- **Reservas** por MAC, con la IP **fuera del pool**.
- **SNAT** para IP pública fija; **MASQUERADE** para IP pública dinámica.
- **Una sola ruta por defecto**: solo la interfaz pública recibe puerta de enlace.

### Chuleta de comandos

| Para… | Comando |
|---|---|
| Ver si Kea escucha | `sudo ss -ulpn \| grep 67` |
| Log de Kea en directo | `sudo journalctl -fu kea-dhcp4-server` |
| Concesiones | `sudo column -s, -t /var/lib/kea/kea-leases4.csv` |
| Capturar DHCP | `sudo tcpdump -i enp3s0 -n -v 'udp port 67 or udp port 68'` |
| Renovar en Fedora | `sudo nmcli con up "Conexión cableada 1"` |
| Ver concesión en Fedora | `nmcli -f DHCP4 con show "Conexión cableada 1"` |
| Renovar en Windows | `ipconfig /release` + `ipconfig /renew` |
| Renovar en Ubuntu | `sudo networkctl renew enp1s0` |
| Reglas NAT | `sudo iptables -t nat -L -n -v` |

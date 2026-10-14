---
title: "Tarea 5: Creación de escenarios con Terraform"
asignatura: ProyIntMod
date: 2026-10-14
resumen: "Creo escenarios de máquinas virtuales en libvirt con Terraform y cloud-init. Comparo una red aislada con dirección frente a una muy aislada sin ella, monto un escenario de tres máquinas con salto por SSH, genero el inventario de Ansible desde Terraform y repaso los errores que me fui encontrando."
tags:
  - Terraform
  - OpenTofu
  - libvirt
  - cloud-init
  - Ansible
---

**Repositorio:** fork de [josedom24/ejercicios_pi](https://github.com/josedom24/ejercicios_pi). Cada ejemplo se hace dentro de su directorio `opentofu/ejemploX`.

## Preparación común

Antes de crear cualquier escenario hay que poner la clave pública del anfitrión en los ficheros `cloud-init/user-dataX.yaml`, sustituyendo la línea de ejemplo:

```yaml
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... sustituye esta línea por tu clave pública
```

La clave se obtiene con:

```bash
cat ~/.ssh/id_ed25519.pub
```

> ⚠️ **Importante:** cloud-init solo aplica esta configuración en el primer arranque. Si la máquina ya se creó sin la clave, hay que hacer `terraform destroy` y `terraform apply` de nuevo.

> Si falla el acceso por SSH, se puede entrar por consola con `virsh console <máquina>`, con el usuario de la imagen (`debian` o `ubuntu`) y la contraseña `asdasd`.

---

## Ejemplo 4: máquina conectada a dos redes (DHCP y estática)

**Directorio:** `opentofu/ejemplo4`

La máquina `ej4-server1` (Debian 13) se conecta a dos redes definidas en `network.tf`:

| Red | Modo | Rango | DHCP | Interfaz en la VM |
|---|---|---|---|---|
| `ej4-nat-dhcp` | `nat` | `192.168.100.0/24` | Sí | `ens3` (DHCP) |
| `ej4-aislada-static` | `none` | `192.168.130.0/24` | No | `ens4` (`192.168.130.10/24`) |

La IP estática se configura en `cloud-init/network-config1.yaml` (netplan). Terraform solo conoce las IP asignadas por DHCP, así que la estática está escrita a mano también en `output.tf` (`ip2`).

> Si se cambia la IP estática, hay que cambiarla en los dos ficheros.

### 4.1. Red aislada con dirección

Creación del escenario:

```bash
cd opentofu/ejemplo4
terraform init
terraform apply
terraform output
```

Dentro de la VM (`ssh debian@<ip1>`), `ip a` muestra `ens3` con una IP `192.168.100.x` y `ens4` con `192.168.130.10/24`.

En el anfitrión, nombre del bridge de la red:

```
$ virsh net-info ej4-aislada-static
Nombre:         ej4-aislada-static
UUID:           5745ef46-ac88-4eaa-8656-0008fd413ec5
Activar:        si
Persistente:    si
Autoinicio:     si
Puente:         virbr3
```

Comprobación:

```bash
ip a show virbr3
ping -c 4 192.168.130.10
```

> ✅ **Resultado: el ping funciona.** La red es aislada (`mode = "none"`): no tiene NAT ni salida al exterior. Pero al tener `addresses = ["192.168.130.0/24"]`, libvirt asigna al bridge `virbr3` del anfitrión la dirección `192.168.130.1/24`. El anfitrión está en la misma red que la VM, tiene ruta hacia ella a través de `virbr3` y llega a `192.168.130.10`.

```bash
terraform destroy
```

### 4.2. Red muy aislada (sin dirección)

Cambios realizados:

**`network.tf`** — nueva red sin `addresses`:

```hcl
# Red muy aislada: sin addresses, el anfitrión no tiene IP en el bridge
resource "libvirt_network" "ej4-muy-aislada" {
  name      = "ej4-muy-aislada"
  mode      = "none"
  dhcp { enabled = false }
  autostart = true
}
```

**`main.tf`** — tercera interfaz en la VM:

```hcl
  # Red 3: muy aislada (IP estática 172.16.0.10)
  network_interface {
    network_id = libvirt_network.ej4-muy-aislada.id
  }
```

**`cloud-init/network-config1.yaml`** — configuración de la nueva interfaz:

```yaml
    ens5:
      dhcp4: false
      addresses: ["172.16.0.10/16"]     # IP estática en red muy aislada
```

**`output.tf`** — IP estática escrita a mano:

```hcl
    ip3    = "172.16.0.10"    # estática: está en cloud-init/network-config1.yaml
```

Comprobación:

```bash
terraform apply
virsh net-info ej4-muy-aislada     # nombre del bridge
ip a show virbrX
ping -c 4 172.16.0.10
```

> ❌ **Resultado: el ping no funciona.** Sin `addresses`, libvirt crea el bridge sin ninguna IP en el anfitrión: en `ip a` aparece el bridge, pero sin línea `inet`. El anfitrión no pertenece a `172.16.0.0/16`, no tiene ruta hacia esa red y envía el ping por su puerta de enlace por defecto (el router), que no conoce esa red. La VM tiene la IP bien configurada, pero por esa red solo podrían comunicarse con ella otras máquinas conectadas al mismo bridge.

```bash
terraform destroy
```

### Diferencia clave

| Tipo de red | Configuración | Salida al exterior | IP del anfitrión en el bridge | El anfitrión llega a las VMs |
|---|---|---|---|---|
| **Aislada** | `mode = "none"` con `addresses` | No | Sí | ✅ Sí |
| **Muy aislada** | `mode = "none"` sin `addresses` | No | No | ❌ No |

---

## Ejemplo 5: escenario con varias máquinas

**Directorio:** `opentofu/ejemplo5`

| Máquina | Sistema | Usuario | Redes | IP |
|---|---|---|---|---|
| `ej5-server1` | Debian | `debian` | `ej5-nat-dhcp` + `ej5-muy-aislada` | DHCP + `10.0.0.1/24` |
| `ej5-server2` | Ubuntu | `ubuntu` | `ej5-muy-aislada` | `10.0.0.2/24`, gateway `10.0.0.1` |
| `ej5-server3` (añadida) | Ubuntu | `ubuntu` | `ej5-muy-aislada` | `10.0.0.3/24`, gateway `10.0.0.1` |

> `ej5-server1` no tiene activado el reenvío de paquetes ni el NAT, así que las máquinas internas no tienen salida al exterior. Por eso en sus `user-data` está comentada la actualización con apt.

### Inventario de Ansible generado por Terraform

- **`inventario.tftpl`**: plantilla del fichero `hosts` con huecos `${...}`.
- **`inventario.tf`**: el recurso `local_file` rellena la plantilla con `templatefile` y escribe el fichero `hosts`. La IP de `ej5-server1` la obtiene del DHCP de libvirt; las estáticas están escritas a mano.
- `local_file` pertenece al provider `hashicorp/local`, por lo que hace falta `terraform init` antes del primer `apply`.

Las máquinas internas llevan `ProxyJump=debian@<ip de ej5-server1>`. La red `ej5-muy-aislada` no tiene dirección en el anfitrión, así que solo se llega a ellas saltando por `ej5-server1`.

### 5.1. Acceso por SSH entre máquinas

> Antes de conectar, comprobar que la clave está cargada en el agente SSH (`ssh-add -l`; si no hay identidades, `ssh-add`).

```bash
terraform init
terraform apply
ssh -A debian@<ip1>              # acceso a ej5-server1 con reenvío del agente
ping -c 4 10.0.0.2               # desde ej5-server1
ssh ubuntu@10.0.0.2              # salto a ej5-server2
```

Con `ssh -A` el agente SSH del anfitrión se reenvía a `ej5-server1`, así que el segundo salto se autentica con la clave privada del anfitrión sin copiarla a ninguna máquina.

### 5.2. Inventario y Ansible

```bash
cat hosts
ansible -i hosts all -m ping
```

> ✅ Todas las máquinas deben responder `SUCCESS` y `"ping": "pong"`. El aviso `[WARNING]` sobre el intérprete de Python es solo informativo.

### 5.3. Tercera máquina en la red `ej5-muy-aislada`

Ficheros nuevos, copiados de los de `ej5-server2`:

```bash
cp cloud-init/user-data2.yaml cloud-init/user-data3.yaml
cp cloud-init/network-config2.yaml cloud-init/network-config3.yaml
```

**`cloud-init/user-data3.yaml`**:

```yaml
hostname: ej5-server3
```

**`cloud-init/network-config3.yaml`**:

```yaml
      addresses: ["10.0.0.3/24"]  # IP estática en red muy aislada
```

**`main.tf`** — recursos de la nueva máquina añadidos al final:

```hcl
##############################################
# server3 — Ubuntu (cliente interno2)
##############################################

# Clon ligero sobre la imagen base Ubuntu
resource "libvirt_volume" "ej5-server3-disk" {
  name             = "ej5-server3.qcow2"
  pool             = var.libvirt_pool_name
  base_volume_name = var.base_image_ubuntu
  base_volume_pool = var.libvirt_pool_name
  format           = "qcow2"
}

# ISO cloud-init con la configuración inicial de server3
resource "libvirt_cloudinit_disk" "ej5-server3-cloudinit" {
  name           = "ej5-server3-cloudinit.iso"
  pool           = var.libvirt_pool_name
  user_data      = file("${path.module}/cloud-init/user-data3.yaml")
  network_config = file("${path.module}/cloud-init/network-config3.yaml")
}

resource "libvirt_domain" "ej5-server3" {
  name   = "ej5-server3"
  memory = 1024
  vcpu   = 2

  # Red única: muy aislada (IP estática 10.0.0.3, gateway 10.0.0.1)
  network_interface {
    network_id = libvirt_network.ej5-muy-aislada.id
  }

  disk { volume_id = libvirt_volume.ej5-server3-disk.id }
  cloudinit = libvirt_cloudinit_disk.ej5-server3-cloudinit.id

  console {
    type        = "pty"
    target_port = "0"
    target_type = "serial"
  }
}
```

**`output.tf`**:

```hcl
# server3: solo red muy aislada con IP estática
output "ej5-server3" {
  value = {
    nombre = "ej5-server3"
    ip1    = "10.0.0.3" # estática: está en cloud-init/network-config3.yaml
  }
}
```

**`inventario.tf`**, dentro de `templatefile`:

```hcl
    ip_server3 = "10.0.0.3" # estática: está en cloud-init/network-config3.yaml
```

**`inventario.tftpl`**:

```
ej5-server3 ansible_host=${ip_server3} ansible_user=ubuntu ansible_ssh_common_args='-o ProxyJump=debian@${ip_server1}'
```

Creación y comprobación:

```bash
terraform validate
terraform plan
terraform apply
```

```
Apply complete! Resources: 12 added, 0 changed, 0 destroyed.

Outputs:

ej5-server1 = {
  "ip1" = "192.168.100.164"
  "ip2" = "10.0.0.1"
  "nombre" = "ej5-server1"
}
ej5-server2 = {
  "ip1" = "10.0.0.2"
  "nombre" = "ej5-server2"
}
ej5-server3 = {
  "ip1" = "10.0.0.3"
  "nombre" = "ej5-server3"
}
```

```bash
cat hosts
ssh -A debian@192.168.100.164
ping -c 4 10.0.0.3                    # desde ej5-server1
ssh ubuntu@10.0.0.3                   # desde ej5-server1
ip a show ens3                        # desde ej5-server3
ansible -i hosts all -m ping          # desde el anfitrión
terraform destroy
```

> ✅ Las tres máquinas responden `pong`. Igual que `ej5-server2`, a `ej5-server3` solo se llega a través de `ej5-server1`, por eso lleva también el `ProxyJump` en el inventario.

---

## Problemas encontrados y soluciones

### `Host key verification failed` en Ansible (máquinas internas)

El anfitrión nunca se había conectado directamente a las máquinas internas, así que no tenía guardada su huella. Ansible no puede mostrar la pregunta de confirmación y falla.

**Solución:** conectarse una vez a mano desde el anfitrión con el mismo salto que usa Ansible y aceptar la huella:

```bash
ssh -J debian@<ip1> ubuntu@10.0.0.2
ssh -J debian@<ip1> ubuntu@10.0.0.3
```

**Alternativa:** `ANSIBLE_HOST_KEY_CHECKING=False ansible -i hosts all -m ping`.

### `Failed to get "write" lock` al crear `ej5-server3`

Al copiar el bloque de `server2`, las líneas `disk` y `cloudinit` de `ej5-server3` seguían apuntando a los recursos de `server2`. La máquina intentaba usar el disco de `ej5-server2`, que ya estaba en uso.

**Solución:** corregir las referencias y comprobar que no queda ninguna:

```hcl
  disk { volume_id = libvirt_volume.ej5-server3-disk.id }
  cloudinit = libvirt_cloudinit_disk.ej5-server3-cloudinit.id
```

```bash
grep -n "server2" main.tf
```

### `ya existe un dominio 'ej5-server3'`

El intento fallido dejó la máquina definida en libvirt, pero Terraform no la tenía registrada en su estado.

**Solución:** borrarla a mano y volver a aplicar:

```bash
virsh undefine ej5-server3
virsh list --all
terraform apply
```

### `Permission denied` en `~/.ssh/known_hosts`

El fichero había quedado como propiedad de `root` (por ejecutar algún comando de SSH con `sudo`), y SSH no podía leerlo.

**Solución:**

```bash
sudo chown gabriel:gabriel ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts
```

Después hay que volver a aceptar las huellas de las máquinas.

### `REMOTE HOST IDENTIFICATION HAS CHANGED`

Libvirt reutiliza IP entre escenarios, y la huella guardada es de una máquina anterior.

**Solución:** `ssh-keygen -R <ip>` y volver a conectar.

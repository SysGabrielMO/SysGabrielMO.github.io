---
title: "Diseño de particionado para instalar Debian 13 con LVM"
asignatura: ASO
date: 2026-09-17
resumen: "Esquema de particionado de un disco de 1 TB para Debian 13 con LVM, justificación de cada volumen y configuración de zswap."
tags:
  - Debian 13
  - LVM
  - Particionado
  - UEFI
  - zswap
---

## 1. Objetivo

Diseñar el esquema de particionado de un disco de **1 TB** para instalar **Debian 13 (Trixie)**, usando **LVM** para las particiones internas del sistema (`/`, `/var`, `/home`, `swap`) y dejando **EFI** y **/boot** fuera del LVM, tal como exige el proceso de arranque UEFI.

Sistema de ficheros elegido para todos los volúmenes: **ext4**.
RAM de la máquina: **16 GB**.

## 2. Esquema de particiones final

### Fuera del LVM

| Partición | Tamaño | Formato | Punto de montaje |
|---|---|---|---|
| EFI | 300 MB | FAT32 (EFI System Partition) | /boot/efi |
| /boot | 1 GB | ext4 | /boot |

### Dentro del LVM (grupo de volúmenes `vg0`, ~998 GB)

| Volumen lógico | Tamaño | Formato | Punto de montaje |
|---|---|---|---|
| `root` | 40 GB | ext4 | / |
| `var` | 80 GB | ext4 | /var |
| `home` | 700 GB | ext4 | /home |
| `swap` | 8 GB | swap | — |
| *(sin asignar)* | ~170 GB | — | reserva para `lvextend` |

![Esquema de particionado](/assets/img/ASO/esquema-particionado.svg)

## 3. Justificación de las decisiones

- **EFI y /boot fuera de LVM**: el firmware UEFI y GRUB necesitan leer estas particiones antes de que el soporte de LVM esté activo, así que no pueden vivir dentro del contenedor lógico.
- **300 MB para EFI**: algo por encima del mínimo (256 MB) como margen de seguridad para entradas de arranque adicionales.
- **`/var` separado de `/`**: aísla logs, cachés y datos variables (incluidos contenedores Docker si se usan en el futuro) para que un `/var` lleno no deje inutilizable todo el sistema.
- **`root` en 40 GB**: con `/var` ya separado, en `/` solo quedan el sistema base y los paquetes, que en Debian suelen rondar 15–25 GB; 40 GB da margen sin desperdiciar espacio.
- **`/var` en 80 GB** (extremo alto del rango 60–80 GB): por el perfil de uso (trabajo IT, posibles contenedores).
- **`/home` con el grueso del disco (700 GB)**: es el espacio que más se va a aprovechar en el uso diario.
- **~170 GB sin asignar en el VG**: no se reparte el 100% del grupo de volúmenes de entrada. Esto permite ampliar cualquier volumen lógico más adelante con `lvextend`, sin tener que reducir otro ni arriesgar datos.
- **Swap de 8 GB (50% de la RAM)**: suficiente como red de seguridad para picos de uso de memoria en uso normal. *Si se quisiera usar hibernación, el swap debería ser igual o mayor que la RAM (≥16 GB).*

## 4. Pasos en el instalador de Debian 13

1. En *"Particionado de discos"*, elegir **Manual**.
2. Seleccionar el disco → crear tabla de particiones nueva (GPT en UEFI).
3. Crear partición **EFI**: 300 MB, al principio, tipo *Partición EFI del sistema*.
4. Crear partición **/boot**: 1 GB, ext4, punto de montaje `/boot`.
5. Crear partición con el resto del disco: tipo *volumen físico para LVM*.
6. Entrar en **Configurar el Gestor de Volúmenes Lógicos (LVM)**:
   - Crear grupo de volúmenes `vg0` sobre la partición LVM.
   - Crear volúmenes lógicos: `root` (40G), `var` (80G), `home` (700G), `swap` (8G).
   - No asignar el resto: se deja libre a propósito.
7. Volver al particionado y asignar a cada volumen lógico su formato y punto de montaje (ext4 + `/`, `/var`, `/home`; swap sin punto de montaje).
8. **Finalizar el particionado y escribir los cambios en el disco**.

## 5. Ampliar un volumen en el futuro

Ejemplo para ampliar `/var` usando parte del espacio libre del VG:

```bash
lvextend -L +20G /dev/vg0/var
resize2fs /dev/vg0/var
```

Esto se puede hacer en caliente, sin desmontar el sistema.

## 6. Zswap: caché comprimida delante del swap

### 6.1 Qué es y por qué añadirlo

`zswap` es una caché de compresión en RAM que se coloca **delante** del dispositivo de swap: cuando el kernel decide expulsar páginas de memoria, primero intenta comprimirlas y guardarlas en esta caché; solo si la caché se llena escribe las páginas (comprimidas) al swap real, en este caso el volumen lógico `swap` (8 GB) del `vg0`. No sustituye al swap definido en el punto 2, lo complementa: reduce la cantidad de I/O que llega al disco bajo presión de memoria y acelera la recuperación de páginas swapeadas, a costa de un poco de uso de CPU para comprimir/descomprimir.

Es distinto de `zram` (que crea un dispositivo de bloque comprimido en RAM y se usa *como* swap): aquí ya existe un swap en LVM sobre disco, así que `zswap` es el complemento natural en lugar de `zram`.

### 6.2 Comprobar que el kernel lo soporta

El kernel de Debian 13 trae `zswap` compilado dentro del propio kernel (no como módulo cargable), así que basta con comprobar que la opción está activada:

```bash
zgrep CONFIG_ZSWAP /proc/config.gz
# o, si no existe /proc/config.gz:
grep CONFIG_ZSWAP /boot/config-$(uname -r)
```

Debe aparecer `CONFIG_ZSWAP=y`. Al estar integrado en el kernel (no como módulo), no se configura desde `/etc/modprobe.d/`: la configuración persistente se hace por parámetros de arranque del kernel (ver 6.4).

### 6.3 Parámetros de zswap

| Parámetro | Valor recomendado | Descripción |
|---|---|---|
| `enabled` | `Y` | Activa la caché de zswap (en Debian suele venir desactivada por defecto). |
| `compressor` | `zstd` | Algoritmo de compresión; buena relación ratio/velocidad en CPUs actuales. |
| `zpool` | `zsmalloc` | Asignador de memoria para las páginas comprimidas; el más eficiente en espacio. |
| `max_pool_percent` | `20` | Porcentaje máximo de la RAM total que puede ocupar la caché comprimida. |

Estos parámetros son visibles y modificables en caliente (para pruebas puntuales, sin persistencia) en `/sys/module/zswap/parameters/`.

### 6.4 Activarlo de forma persistente (fichero de configuración)

Al ser `zswap` parte integrada del kernel, el fichero de configuración persistente es el de parámetros de arranque de GRUB: **`/etc/default/grub`**.

1. Editar `/etc/default/grub` y añadir los parámetros a la línea `GRUB_CMDLINE_LINUX_DEFAULT`:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet zswap.enabled=1 zswap.compressor=zstd zswap.zpool=zsmalloc zswap.max_pool_percent=20"
```

2. Regenerar la configuración de GRUB:

```bash
update-grub
```

3. Reiniciar el sistema para que el kernel arranque con estos parámetros.

### 6.5 Verificar tras el reinicio

```bash
cat /sys/module/zswap/parameters/enabled          # Y
cat /sys/module/zswap/parameters/compressor       # zstd
cat /sys/module/zswap/parameters/zpool            # zsmalloc
cat /sys/module/zswap/parameters/max_pool_percent # 20
dmesg | grep -i zswap
```

### 6.6 Justificación de las decisiones

- **`zswap` en vez de (o además de) ampliar el swap**: con 16 GB de RAM, comprimir páginas antes de escribirlas a disco reduce la latencia percibida bajo presión de memoria, sin tocar el esquema de particionado ya definido.
- **`compressor=zstd`**: mejor ratio de compresión que `lzo`/`lz4` con un coste de CPU asumible en hardware actual; prioriza ahorrar I/O a disco frente a mínimo uso de CPU.
- **`zpool=zsmalloc`**: es el asignador más compacto de los disponibles (`zbud`, `z3fold`, `zsmalloc`), aprovecha mejor la RAM reservada para la caché.
- **`max_pool_percent=20`**: valor por defecto del kernel; limita la caché comprimida a ~3,2 GB de los 16 GB de RAM, dejando margen suficiente para el resto de procesos.
- **Configuración vía GRUB y no vía módulo**: al estar `zswap` compilado dentro del kernel (no como `.ko`), no existe un fichero en `/etc/modprobe.d/` equivalente; los parámetros de arranque en `/etc/default/grub` son el único mecanismo persistente entre reinicios.

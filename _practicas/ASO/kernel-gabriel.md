---
title: "Compilación del kernel 7.2.6 (trixie-backports) en Debian 13"
asignatura: ASO
date: 2026-10-12
resumen: "Compilo el kernel 7.2.6 de trixie-backports con localmodconfig para incluir solo los módulos que usa mi equipo, lo empaqueto con bindeb-pkg y lo instalo. Compruebo que todo el hardware funciona, comparo su tamaño con el kernel de Debian (277 módulos frente a 4230) y lo arranco directamente desde la UEFI con EFI stub, sin GRUB."
tags:
  - Kernel
  - Compilación desde fuentes
  - EFI stub
  - efibootmgr
  - Debian
---

## 1. Preparación

Añadí `trixie-backports` en `/etc/apt/sources.list.d/backports.sources` e instalé el código fuente y las herramientas:

```bash
sudo apt install -t trixie-backports linux-source
sudo apt install build-essential bc kmod cpio flex bison libncurses-dev \
  libelf-dev libssl-dev dwarves rsync dpkg-dev debhelper efibootmgr
mkdir ~/kernel && cd ~/kernel
tar xaf /usr/src/linux-source-7.2.tar.xz && cd linux-source-7.2
```

## 2. Configuración

Antes de configurar puse en uso todo mi hardware (wifi, bluetooth, webcam, adaptador Ethernet USB-C) y arranqué una VM de KVM, para que `localmodconfig` incluyera también los módulos de virtualización.

```bash
lsmod > ~/kernel/modulos_usados.txt
cp /boot/config-$(uname -r) .config
yes "" | make localmodconfig
scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
scripts/config --set-str SYSTEM_REVOCATION_KEYS ""
scripts/config --disable DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT
scripts/config --enable DEBUG_INFO_NONE
scripts/config --set-str LOCALVERSION "-gabriel"
scripts/config --enable CMDLINE_BOOL
scripts/config --set-str CMDLINE "root=/dev/mapper/vg_debian-root ro quiet zswap.enabled=1"
make olddefconfig
```

- **`TRUSTED_KEYS` / `REVOCATION_KEYS` vacíos:** la config de Debian apunta a certificados que no tengo y la compilación fallaría.
- **`DEBUG_INFO_NONE`:** compila más rápido y no genera un paquete de depuración enorme.
- **`CMDLINE`:** los parámetros de arranque van dentro del kernel, necesario para arrancar sin GRUB.

## 3. Compilación e instalación

```bash
make -j$(nproc) bindeb-pkg
cd ~/kernel
sudo dpkg -i linux-image-7.2.6-gabriel_7.2.6-1_amd64.deb linux-headers-7.2.6-gabriel_7.2.6-1_amd64.deb
```

Tardó **15 minutos**. Al instalar se generó el initrd y se actualizó GRUB.<img src="/assets/img/ASO/2026-10-12-19-56-27-image.png" title="" alt="" width="703">

## 4. Comprobaciones

Todo el hardware funciona: gráfica (`amdgpu`), wifi (`rtw89_8852be`), audio, bluetooth, webcam, Ethernet USB (`r8152`) y máquinas virtuales.

![](/assets/img/ASO/2026-10-12-19-56-56-image.png)

![](/assets/img/ASO/2026-10-12-19-57-27-image.png)

### Comparación de tamaño con el kernel de Debian

```bash
ls -lh /boot/vmlinuz-* /boot/initrd.img-*
du -sh /lib/modules/6.12.111+deb13-amd64 /lib/modules/7.2.6-gabriel
find /lib/modules/6.12.111+deb13-amd64 -name "*.ko*" | wc -l
find /lib/modules/7.2.6-gabriel -name "*.ko*" | wc -l
dpkg-query -W -f='${Package}\t${Installed-Size} KB\n' 'linux-image-6.12.111*' 'linux-image-7.2.6-gabriel'
```

|                          | Debian 6.12.111 | Mío 7.2.6-gabriel |
| ------------------------ | --------------- | ----------------- |
| Imagen (`vmlinuz`)       | 12 MB           | 11 MB             |
| `initrd.img`             | 71 MB           | **43 MB**         |
| Módulos (`/lib/modules`) | 107 MB          | **15 MB**         |
| Número de módulos        | 4230            | **277**           |
| Paquete instalado        | 106 MB          | **32 MB**         |

La imagen del kernel apenas cambia, porque el núcleo base es parecido en los dos. La gran diferencia está en los módulos: el mío tiene unos **15 veces menos**, porque solo incluye los que usa mi equipo. Por eso también el initrd es más pequeño.

![](/assets/img/ASO/2026-10-12-20-04-09-image.png)

## 5. Secure Boot

`mokutil --sb-state` devuelve `SecureBoot disabled`, así que no fue necesario firmar el kernel.

## 6. Arranque sin gestor de arranque

El kernel tiene `CONFIG_EFI_STUB=y`, así que la UEFI puede arrancarlo directamente. Copié kernel e initrd a la partición EFI y creé una entrada:

```bash
sudo mkdir -p /boot/efi/EFI/gabriel
sudo cp /boot/vmlinuz-7.2.6-gabriel /boot/efi/EFI/gabriel/vmlinuz.efi
sudo cp /boot/initrd.img-7.2.6-gabriel /boot/efi/EFI/gabriel/initrd.img
sudo efibootmgr -C --disk /dev/nvme0n1 --part 1 --label "Debian kernel Gabriel" \
  --loader '\EFI\gabriel\vmlinuz.efi' --unicode 'initrd=\EFI\gabriel\initrd.img'
sudo efibootmgr -n 0005 && sudo reboot
```

Tras arrancar, `/proc/cmdline` no contiene `BOOT_IMAGE=` (lo añade GRUB) y `efibootmgr` muestra `BootCurrent: 0005`, así que arrancó sin GRUB:

```
root=/dev/mapper/vg_debian-root ro quiet zswap.enabled=1 initrd=\EFI\gabriel\initrd.img
```

![](/assets/img/ASO/2026-10-12-19-59-30-image.png)

![](/assets/img/ASO/2026-10-12-19-59-47-image.png)

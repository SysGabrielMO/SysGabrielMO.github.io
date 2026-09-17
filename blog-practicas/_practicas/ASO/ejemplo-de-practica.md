---
title: Ejemplo de practica
asignatura: ASO
date: 2026-09-17
resumen: Práctica de ejemplo para comprobar que el sitio se ve bien. Bórrala cuando publiques las tuyas.
tags:
  - Debian
  - LVM
---

## Objetivo

Comprobar cómo se ven en el blog los distintos elementos de una práctica:
encabezados, listas, bloques de código, tablas e imágenes.

## Desarrollo

Un bloque de código con el aspecto que tendrán tus comandos:

```bash
sudo apt update
sudo apt install lvm2
sudo pvcreate /dev/sda3
```

Una lista de pasos:

1. Crear el volumen físico sobre la partición.
2. Crear el grupo de volúmenes.
3. Repartir el espacio entre los volúmenes lógicos.

Y una tabla:

| Punto de montaje | Tamaño | Sistema de ficheros |
|---|---|---|
| `/` | 40 GB | ext4 |
| `/var` | 100 GB | ext4 |
| `/home` | 800 GB | ext4 |

> Los comentarios o advertencias se ven así.

## Conclusiones

Si esta página se ve con el menú arriba, la cabecera y el texto bien
espaciado, el sitio está correctamente instalado.

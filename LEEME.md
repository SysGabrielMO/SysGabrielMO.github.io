# Cuaderno de prácticas

Sitio estático (Jekyll + GitHub Pages) con las prácticas de 2º de ASIR.

**La fuente de la verdad son los markdown de `~/Documentos/practicas`.**
Este repositorio es una copia publicada de ese contenido; no edites aquí las
prácticas, edítalas allí.

## Estructura

| Ruta | Para qué sirve |
|---|---|
| `_config.yml` | Título, autor, centro, curso y usuario de GitHub. |
| `_data/asignaturas.yml` | Lista de asignaturas: código de carpeta, nombre visible y descripción. El orden de la lista es el orden en la web. |
| `index.html` | Portada. |
| `asignaturas.html` | Página con todas las asignaturas y sus prácticas. |
| `_layouts/` | Plantilla base (menú y pie) y plantilla de práctica. |
| `_practicas/<codigo>/` | Los markdown ya preparados para publicar. |
| `assets/css/estilo.css` | Todo el diseño. |
| `assets/img/<codigo>/` | Imágenes de las prácticas. |

## Cabecera de cada práctica

Los ficheros publicados llevan delante un bloque como este:

```
---
title: Memoria particionado debian13
asignatura: ASO
date: 2026-09-17
resumen: Una o dos frases que salen en la portada.
tags:
  - Debian
  - LVM
---
```

- `title`: el nombre del fichero de origen, con los guiones convertidos en
  espacios y la primera letra en mayúscula.
- `asignatura`: el código de la carpeta de origen.
- `date`: la fecha de modificación del fichero. Marca el orden.
- `resumen` y `tags` son opcionales.

Las imágenes se copian a `assets/img/<codigo>/` y los enlaces del markdown se
reescriben a esa ruta.

## Publicar a mano

```bash
git add .
git commit -m "Nuevas prácticas"
git push
```

El sitio tarda un par de minutos en actualizarse.

## Ver el sitio en local (opcional)

```bash
sudo apt install ruby-full build-essential
gem install --user-install bundler jekyll
jekyll serve
```

Queda en http://localhost:4000

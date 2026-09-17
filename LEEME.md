# Cuaderno de prácticas

Sitio estático (Jekyll + GitHub Pages) que publica las prácticas de 2º de ASIR.

La fuente de la verdad son los markdown de `~/Documentos/practicas`. Este
repositorio es solo una copia publicada de ese contenido.

## Estructura

- `_config.yml` — título, subtítulo, autor y curso.
- `_data/asignaturas.yml` — lista de asignaturas. `codigo` es el nombre de la
  carpeta, `nombre` es lo que se ve en la web, y el orden de la lista es el
  orden del índice.
- `_practicas/<codigo>/` — una carpeta por asignatura con los markdown ya
  preparados para publicar.
- `_layouts/` — plantilla base y plantilla de práctica.
- `assets/css/estilo.css` — diseño.
- `assets/img/<codigo>/` — imágenes de las prácticas.

## Cabecera de cada práctica

Los ficheros publicados llevan delante un bloque como este:

```
---
title: Memoria particionado debian13
asignatura: ASO
date: 2026-09-17
---
```

- `title` sale del nombre del fichero, con los guiones convertidos en espacios.
- `asignatura` es el código de la carpeta de origen.
- `date` es la fecha de modificación del fichero, y marca el orden dentro de
  la asignatura.

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

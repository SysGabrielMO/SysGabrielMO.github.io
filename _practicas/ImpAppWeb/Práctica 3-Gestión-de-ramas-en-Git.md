---
title: "Práctica 3. Gestión de ramas en Git"
asignatura: ImpAppWeb
date: 2026-10-01
resumen: "Gestiono el ciclo de vida de las ramas en Git: creo la rama primera y la fusiono en main sin conflicto, provoco a propósito un conflicto editando la misma línea desde la rama segunda, lo resuelvo a mano y subo las dos ramas a GitHub."
tags:
  - Git
  - Ramas
  - Merge
  - Conflictos
  - GitHub
---

**Gabriel Merencio Ortega** · 2º ASIR

En esta práctica vamos a aprender a gestionar el ciclo de vida de las ramas (creación, modificación, borrado, ...), su unión y la resolución de los conflictos que pueden surgir al unirlas. Para ello usaremos el repositorio de la **práctica 1**, que ya está enlazado a su remoto en GitHub.

## Enlaces

- Repositorio: [practica1gabriel](https://github.com/SysGabrielMO/practica1gabriel)
- Rama `segunda`: [practica1gabriel/tree/segunda](https://github.com/SysGabrielMO/practica1gabriel/tree/segunda)

## 0. Punto de partida

Entramos en el repositorio y comprobamos que estamos en `main`, sin cambios pendientes y sincronizados con el remoto:

```bash
git status
git pull
```

![](/assets/img/ImpAppWeb/2026-10-01-12-25-38-image.png)

## 1. Creación de ramas

Creamos la rama `primera` y comprobamos que existe:

```bash
git branch primera
git branch
```

![](/assets/img/ImpAppWeb/2026-10-01-12-26-22-image.png)

`git branch` muestra todas las ramas locales. La rama en la que estamos aparece marcada con `*`; en este caso seguimos en `main`, porque `git branch` crea la rama pero no nos cambia a ella.

## 2. Trabajar con la rama

Nos cambiamos a la rama `primera`, creamos un fichero nuevo y hacemos commit:

```bash
git checkout primera
nano ramas.txt
cat ramas.txt
git add .
git commit -m "Añadido ramas.txt en la rama primera"
```

Contenido del fichero `ramas.txt`:

```
esta es la primera vez que creo una Rama en git
```

![](/assets/img/ImpAppWeb/2026-10-01-12-29-16-image.png)

![](/assets/img/ImpAppWeb/2026-10-01-12-29-38-image.png)

Volvemos a la rama principal y fusionamos `primera` en ella:

```bash
git checkout main
git merge primera
git log --oneline --graph --all
```

Salida del `git log`:

```
* 9f06ad7 (HEAD -> main, primera) Añadido ramas.txt en la rama primera
* 4da4b39 (origin/main) Renombrado estilos.css a styles.css
* ae65d3d He añadido footer a la pagina princial
* 0bf1beb Añadido .gitignore para excluir notas.txt
* c49192f Estructura inicial del sitio web
```

![](/assets/img/ImpAppWeb/2026-10-01-12-31-55-image.png)

**¿Se ha producido conflicto?**

No. Desde que creamos la rama `primera`, en `main` no se ha hecho ningún commit, así que Git no tiene nada que mezclar: simplemente adelanta el puntero de `main` hasta el último commit de `primera`.

Además, `ramas.txt` es un fichero nuevo que no existía en `main`. Los conflictos solo aparecen cuando la **misma parte** de un fichero se ha modificado en las dos ramas.

## 3. Borrar la rama primera

```bash
git branch -d primera
git branch
```

![](/assets/img/ImpAppWeb/2026-10-01-12-34-25-image.png)

## 4. Provocar un conflicto

Creamos la rama `segunda`, nos cambiamos a ella y modificamos `ramas.txt`:

```bash
git checkout -b segunda
printf "Linea modificada en la rama segunda\n" > ramas.txt
git add ramas.txt
git commit -m "Modificado ramas.txt en segunda"
```

![](/assets/img/ImpAppWeb/2026-10-01-12-36-55-image.png)

Volvemos a `main` y modificamos **la misma línea** del mismo fichero con un texto distinto:

```bash
git checkout main
printf "Linea modificada en la rama main\n" > ramas.txt
git add ramas.txt
git commit -m "Modificado ramas.txt en main"
```

![](/assets/img/ImpAppWeb/2026-10-01-12-38-56-image.png)

Intentamos fusionar `segunda` en `main`:

```bash
git merge segunda
git status
```

![](/assets/img/ImpAppWeb/2026-10-01-12-47-20-image.png)

Git detecta el conflicto y detiene la fusión. El contenido del fichero donde se ha producido es:

```bash
cat ramas.txt
```

```
<<<<<<< HEAD
Linea modificada en la rama main
=======
Linea modificada en la rama segunda
>>>>>>> segunda
```

![](/assets/img/ImpAppWeb/2026-10-01-12-48-08-image.png)

## 5. Resolución de conflictos

Editamos el fichero, dejamos el contenido final y **eliminamos las marcas** `<<<<<<<`, `=======` y `>>>>>>>`:

```bash
nano ramas.txt
```

![](/assets/img/ImpAppWeb/2026-10-01-12-49-08-image.png)

He decidido conservar los cambios de las dos ramas, así que el contenido final del fichero es:

```
Linea modificada en la rama main
Linea modificada en la rama segunda
```

Marcamos el conflicto como resuelto y cerramos la fusión con un commit:

```bash
git add ramas.txt
git commit -m "Resuelto conflicto en ramas.txt"
git log --oneline --graph --all
```

![](/assets/img/ImpAppWeb/2026-10-01-12-51-40-image.png)

En el historial de commits se ve cómo las dos ramas se separan y vuelven a unirse en el commit de merge.

### 5.1. Sincronizar la rama segunda en el remoto

La resolución solo está en `main`, así que primero actualizamos `segunda` con ella y después subimos las dos ramas a GitHub:

```bash
git checkout segunda
git merge main
git push origin main
git push origin segunda
git checkout main
git branch -a
```

![](/assets/img/ImpAppWeb/2026-10-01-12-54-28-image.png)

En la salida de `git branch -a` aparece `remotes/origin/segunda`. Las ramas no se crean automáticamente en GitHub: es necesario hacer `push` de cada una. Las ramas remotas aparecen en rojo para distinguirlas de las locales.

![](/assets/img/ImpAppWeb/2026-10-01-12-55-05-image.png)

Vamos a GitHub a comprobar que está todo subido correctamente:

![](/assets/img/ImpAppWeb/2026-10-01-12-56-51-image.png)

![](/assets/img/ImpAppWeb/2026-10-01-12-57-06-image.png)

![](/assets/img/ImpAppWeb/2026-10-01-12-57-26-image.png)

Observamos que está el fichero nuevo con el último commit que hemos hecho al resolver el conflicto. Además, aparece la rama `segunda`, que hemos subido con `push`.

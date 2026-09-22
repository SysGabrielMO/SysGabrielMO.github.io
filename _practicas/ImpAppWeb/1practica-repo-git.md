---
title: "Práctica 1. Creación Repositorio Git"
asignatura: ImpAppWeb
date: 2026-09-22
resumen: "Creo mi primer repositorio Git desde cero: lo inicializo, hago commits, ignoro archivos con .gitignore, comparo cambios con git diff, renombro y borro archivos con git mv y git rm, y deshago un commit erróneo con git revert y git reset."
tags:
  - Git
  - Control de versiones
  - gitignore
  - git revert
  - git reset
---

Gabriel Merencio Ortega 2º ASIR 2026/27 

En esta práctica, vamos a configurar nuestro primer repositorio Git.

## 0. Configuración previa

Antes de empezar nuestra práctica, debemos hacer una configuración previa para identificarnos en Git. En mi caso ya lo hice en su tiempo, pero se hace de esta manera:

```bash
git config --global user.name "Gabriel Merencio Ortega"
git config --global user.email gabrielbusiness2007@gmail.com
git config --list
```

![](/assets/img/ImpAppWeb/2026-09-22-12-09-57-image.png)

## 1. Inicialización del repositorio

Empezaremos creando nuestra nueva carpeta y convirtiéndola en un repositorio.

Lo realizaremos de la siguiente manera:

```bash
mkdir gabriel_merencio
cd gabriel_merencio
git init
```

![](/assets/img/ImpAppWeb/2026-09-22-12-12-24-image.png)

Aparte de que se ve visualmente si leemos el mensaje, si queremos ver si se ha creado correctamente ejecutaremos el comando `ls -la` para que nos muestre que se ha creado la carpeta `.git`. Esta carpeta es donde Git guarda todo el historial del repositorio.

## 2. Primeros archivos

Creamos nuestros primeros archivos como bien indica la práctica:

![](/assets/img/ImpAppWeb/2026-09-22-12-16-17-image.png)

## 3. Estado del repositorio

Una vez creados los archivos nuevos, vamos a comprobar el estado del repo antes de subir nada.

Esto lo haremos con:

```bash
git status
```

![](/assets/img/ImpAppWeb/2026-09-22-12-17-13-image.png)

Como vemos, no hemos añadido nada aún, por eso nos sale **archivos sin seguimiento**.

## 4. Primer commit

Vamos a añadir nuestro primer commit solo añadiendo `index.html` y `estilos.css`.

```bash
git add index.html estilos.css
git status
```

![](/assets/img/ImpAppWeb/2026-09-22-12-19-07-image.png)

Vamos a realizar el commit:

```bash
git commit -m "Estructura inicial del sitio web"
```

![](/assets/img/ImpAppWeb/2026-09-22-12-20-00-image.png)

Como vemos, nos da la salida del mensaje de que dos archivos han sido cambiados (añadidos en este caso porque eran nuevos) al repositorio.

## 5. Ignorar archivos

> **¿Por qué ignorar archivos?**
> 
> Creo que es una buena práctica a la hora de tener en nuestra carpeta archivos sensibles o apuntes que solo nosotros queramos ver. Como por ejemplo: contraseñas, las claves API de algún lado, ...

Para configurar el `.gitignore` debemos hacer lo siguiente:

Crear el archivo `.gitignore`:

```bash
nano .gitignore
```

Dentro de él escribiremos el nombre del archivo que queremos que no se vea. (Voy a hacerlo con `printf` para que sea mejor visualmente).

![](/assets/img/ImpAppWeb/2026-09-22-12-24-54-image.png)

He realizado un commit para tener más controladas las versiones del repositorio:

![](/assets/img/ImpAppWeb/2026-09-22-12-29-36-image.png)

## 6. Modificación y comparación

Seguiremos modificando los archivos, en este caso vamos a añadir un footer en `index.html`. Para ver qué ha cambiado en nuestro archivo utilizaremos el siguiente comando:

```bash
git diff HEAD
```

![](/assets/img/ImpAppWeb/2026-09-22-12-32-01-image.png)

Como vemos, la línea nueva que hemos añadido sale con un `+` delante y en verde, así tenemos controlado de una manera más sencilla lo que añadimos.

## 7. Segundo commit

Vamos a realizar el segundo commit con nuestra nueva versión del repositorio:

```bash
git add index.html
git commit -m "Añadido footer a la página principal"
```

![](/assets/img/ImpAppWeb/2026-09-22-12-34-22-image.png)

## 8. Renombrar un archivo

Para renombrar un archivo sin borrar el que ya teníamos antes, realizaremos lo siguiente:

```bash
git mv estilos.css styles.css
git status
```

![](/assets/img/ImpAppWeb/2026-09-22-12-35-48-image.png)

> **¿Por qué lo hacemos con `git mv`?**
> 
> Lo hacemos con este comando ya que, si lo hacemos con `mv`, Git lo interpreta como que el archivo se ha eliminado y que se ha creado uno nuevo.

Hacemos el commit:

```bash
git commit -m "Renombrado estilos.css a styles.css"
```

![](/assets/img/ImpAppWeb/2026-09-22-12-37-02-image.png)

## 9. Eliminar un archivo

Vamos a eliminar el archivo `index.html` ya que no es necesario. Lo debemos borrar tanto del sistema como del repo usando el comando de Git adecuado.

Para ello:

```bash
git rm index.html
git status
git commit -m "Eliminado index.html"
```

![](/assets/img/ImpAppWeb/2026-09-22-12-38-52-image.png)

## 10. Un commit erróneo

Hemos borrado `index.html` y ahora queremos volver al último commit. Para ello podríamos hacerlo de la siguiente manera:

Primero vamos a ver el historial de commits:

```bash
git log --oneline
```

![](/assets/img/ImpAppWeb/2026-09-22-12-42-57-image.png)

Tenemos dos formas de volver al último commit.

### Forma 1: `git revert`

Básicamente lo que hace es crear un commit nuevo que hace lo contrario al anterior, es decir, este creará otra vez el archivo:

```bash
git revert HEAD
```

![](/assets/img/ImpAppWeb/2026-09-22-12-44-55-image.png)

Git nos abre el editor para confirmar el mensaje y, como vemos, vuelve el archivo:

![](/assets/img/ImpAppWeb/2026-09-22-12-45-29-image.png)

### Forma 2: `git reset`

Básicamente es volver a antes del último commit realizado. Como hemos realizado antes el revert, en vez de 1 serán 2 los saltos que tendremos que mover para llegar al commit:

```bash
git reset --hard HEAD~2
git log --oneline
```

![](/assets/img/ImpAppWeb/2026-09-22-12-49-39-image.png)

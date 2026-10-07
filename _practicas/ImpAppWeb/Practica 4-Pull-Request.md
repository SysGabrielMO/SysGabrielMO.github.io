---
title: "Práctica 4. Colaboración en GitHub con Pull Requests"
asignatura: ImpAppWeb
date: 2026-10-02
resumen: "Colaboro en un repositorio ajeno con el flujo de fork y Pull Request: hago un fork del repositorio del profesor, aporto mi fichero desde una rama propia y lo propongo con un Pull Request. Después sincronizo mi fork con upstream para traer los cambios de los compañeros y termino haciendo y recibiendo un Pull Request entre compañeros."
tags:
  - Git
  - GitHub
  - Pull Request
  - Fork
  - Upstream
---

**Gabriel Merencio Ortega** · 2º ASIR

En esta práctica vamos a conocer la metodología para colaborar en proyectos alojados en GitHub mediante **Pull Requests**. Para ello haremos un fork del repositorio del profesor, propondremos nuestros cambios con un Pull Request, sincronizaremos nuestro fork con los cambios de todos los compañeros y, por último, haremos y recibiremos un Pull Request entre compañeros.

## Enlaces

- Repositorio del profesor: [Javier-IESGonzaloNazareno/prueba-pr-asir](https://github.com/Javier-IESGonzaloNazareno/prueba-pr-asir)
- Mi fork: [SysGabrielMO/prueba-pr-asir](https://github.com/SysGabrielMO/prueba-pr-asir)
- Mi Pull Request al profesor: [Pull Request #___](https://github.com/Javier-IESGonzaloNazareno/prueba-pr-asir/pull/___)
- Mi fichero en el repositorio del profesor: [files/gmo.md](https://github.com/Javier-IESGonzaloNazareno/prueba-pr-asir/blob/main/files/gmo.md)
- Repositorio del compañero: [<usuario_compañero>/<repo>](https://github.com/%3Cusuario_compa%C3%B1ero%3E/%3Crepo%3E)
- Mi Pull Request al compañero: [Pull Request #___](https://github.com/%3Cusuario_compa%C3%B1ero%3E/%3Crepo%3E/pull/___)
- Pull Request recibido en mi repositorio: [Pull Request #___](https://github.com/SysGabrielMO/%3Crepo%3E/pull/___)



## 1. Fork del repositorio del profesor

Como no tenemos permisos de escritura en el repositorio del profesor, desde la web de GitHub pulsamos el botón **Fork**. Así se crea una copia del repositorio en nuestra cuenta, sobre la que sí podemos trabajar.

![](/assets/img/ImpAppWeb/2026-10-02-12-09-33-image.png) 



En nuestro fork aparece el texto *forked from Javier-IESGonzaloNazareno/prueba-pr-asir*, que indica de qué repositorio procede.

![](/assets/img/ImpAppWeb/2026-10-02-12-10-05-image.png)

## 2. Clonar el fork

Clonamos **nuestro fork** (no el repositorio del profesor) y comprobamos a qué remoto apunta:



```bash
git clone git@github.com:SysGabrielMO/prueba-pr-asir.git
cd prueba-pr-asir
git remote -v
ls files/
```

![](/assets/img/ImpAppWeb/2026-10-02-12-11-14-image.png)

## 3. Crear una rama de trabajo

Creamos la rama `gmo` y nos cambiamos a ella. Trabajamos en una rama para dejar `main` limpio y poder sincronizarlo después con el repositorio del profesor:

bash

```bash
git checkout -b gmo
git branch
```

![](/assets/img/ImpAppWeb/2026-10-02-12-12-04-image.png)



## 4. Modificar el README.md

Editamos el `README.md` y añadimos nuestra línea al final de la lista:

```bash
nano README.md
```

Línea añadida:

```
- [Gabriel Merencio Ortega](https://github.com/Javier-IESGonzaloNazareno/prueba-pr-asir/blob/main/files/gmo.md)
```

![](/assets/img/ImpAppWeb/2026-10-02-12-15-27-image.png)

El enlace apunta al repositorio **del profesor** y no a nuestro fork, porque es ahí donde estará el fichero cuando se acepte el Pull Request.

## 5. Crear el fichero gmo.md

Creamos en el directorio `files` el fichero con nuestras iniciales:

```bash
nano files/gmo.md
cat files/gmo.md
```

Contenido del fichero `gmo.md`:

```
# ¿Qué asignatura me gusta más?

Mi asignatura favorita es **Servicios de Red e Internet**.

## ¿Por qué?

Me gusta porque:

1. Aprendo cómo funcionan por dentro servicios que uso todos los días, como la web, el DNS o el DHCP.
2. Es muy práctica: montamos y configuramos los servicios nosotros mismos en máquinas virtuales.
3. Es de lo que más se usa en el trabajo de un administrador de sistemas.

> Pasar de usar un servicio a saber montarlo y arreglarlo cuando falla es lo que más me motiva.

Algo que he aprendido en ella es el comando `systemctl`, que sirve para gestionar los servicios del sis>

```bash
sudo systemctl restart apache2
sudo systemctl status apache2
```

*Es la asignatura en la que más siento que aprendo cosas útiles para mi futuro.*

Más información: [Documentación de Apache](https://httpd.apache.org/docs/2.4/es/)
```

## 6. Commit y push de los cambios

Revisamos los cambios, los añadimos y hacemos commit con un mensaje significativo:

```bash
git status
git diff README.md
git add README.md files/gmo.md
git commit -m "Añade files/gmo.md con mi asignatura favorita y enlace en el README"
git log --oneline -3
```

![](/assets/img/ImpAppWeb/2026-10-02-12-20-24-image.png)Subimos la rama `gmo` a nuestro fork:

```bash
git push origin gmo
```

![](/assets/img/ImpAppWeb/2026-10-02-12-20-53-image.png)## 7. Crear el Pull Request

En GitHub, desde nuestro fork, pulsamos **Compare & pull request** y comprobamos que la dirección del Pull Request es la correcta:

- **base repository:** `Javier-IESGonzaloNazareno/prueba-pr-asir`, rama `main`
- **head repository:** `SysGabrielMO/prueba-pr-asir`, rama `gmo`

Ponemos como título `Añade fichero gmo.md (Gabriel Merencio Ortega)` y en la descripción explicamos qué cambios proponemos.



El Pull Request queda en estado **Open** a la espera de que el profesor lo revise.

![](/assets/img/ImpAppWeb/2026-10-02-12-21-54-image.png)

## 8. Pull Request aceptado

Cuando el profesor acepta el Pull Request, su estado cambia a **Merged**:

Comprobamos que nuestro enlace aparece en el `README.md` del repositorio del profesor y que el fichero `files/gmo.md` está en su repositorio:

## 9. Sincronizar el repositorio

Nuestro fork no se actualiza solo con los cambios de los compañeros. Para traerlos, añadimos el repositorio del profesor como un segundo remoto llamado `upstream`:



```bash
git remote add upstream https://github.com/Javier-IESGonzaloNazareno/prueba-pr-asir.git
git remote -v
```

![](/assets/img/ImpAppWeb/2026-10-02-12-23-08-image.png)Volvemos a `main`, descargamos los cambios de `upstream`, los fusionamos y los subimos a nuestro fork:



```bash
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
ls files/
git log --oneline -5
```

![](/assets/img/ImpAppWeb/2026-10-02-12-24-05-image.png)En el directorio `files` aparecen ahora los ficheros de todos los compañeros.

Como la rama `gmo` ya está fusionada, la borramos en local y en el remoto:

```bash
git branch -D gmo
git push origin --delete gmo
git branch -a
```

![](/assets/img/ImpAppWeb/2026-10-02-12-25-07-image.png)Comprobamos en GitHub que nuestro fork está al día con el del profesor:



## 10. Pull Request a un compañero

Elegimos al compañero **<Nombre Apellidos>** y su repositorio [<repo>](https://github.com/%3Cusuario_compa%C3%B1ero%3E/%3Crepo%3E). Hacemos **Fork** desde la web:

Mostrar imagen

Clonamos nuestro fork, creamos una rama, hacemos el cambio y lo subimos:

bash

```bash
git clone git@github.com:SysGabrielMO/<repo>.git
cd <repo>
git checkout -b mejora-gmo
nano <fichero>
git diff
git add <fichero>
git commit -m "<mensaje significativo>"
git push origin mejora-gmo
```

Mostrar imagen

El cambio que proponemos es: <describe qué has cambiado y por qué>.

Abrimos el Pull Request desde nuestro fork hacia el repositorio del compañero (`main` ← `mejora-gmo`):

Mostrar imagen

Mostrar imagen

## 11. Pull Request recibido de un compañero

El compañero **<Nombre Apellidos>** hace un Pull Request sobre nuestro repositorio [<repo>](https://github.com/SysGabrielMO/%3Crepo%3E). Lo revisamos en la pestaña **Files changed**:

Mostrar imagen

Si los cambios son correctos, lo aceptamos con **Merge pull request** → **Confirm merge**:

Mostrar imagen

Actualizamos nuestra copia local para tener el cambio del compañero:

bash

```bash
git checkout main
git pull origin main
git log --oneline -3
```

Mostrar imagen

En el historial aparece el commit del compañero junto con el commit de merge del Pull Request.

---
title: "Práctica 2. Trabajando con GitHub"
asignatura: ImpAppWeb
date: 2026-09-24
resumen: "Conecto mis repositorios locales con GitHub: genero y registro la clave SSH, creo y clono un repositorio remoto, subo una segunda práctica renombrando la rama a main, y doy formato al README con Markdown."
tags:
  - Git
  - GitHub
  - SSH
  - Markdown
  - Repositorios remotos
---

En esta práctica vamos a trabajar con GitHub, conectando nuestros repositorios locales con repositorios remotos mediante SSH.

## 1. Configuración de GitHub

### 1.1. Crear la cuenta

Lo primero es crear una cuenta en [GitHub](https://github.com) si todavía no la tenemos.

### 1.2. Comprobar si ya tenemos claves SSH

Antes de hacer nada, comprobamos si ya tenemos un par de claves creado:

```bash
ls -la ~/.ssh
```

Si aparecen los ficheros `id_rsa` (clave privada) e `id_rsa.pub` (clave pública), ya las tenemos.

![](/assets/img/ImpAppWeb/2026-09-24-11-09-18-image.png)

### 1.4. Copiar la clave pública a GitHub

Mostramos el contenido de la **clave pública** y lo copiamos entero:

```bash
cat ~/.ssh/id_rsa.pub
```

> **Importante:** solo se copia el fichero `.pub`. La clave privada (`id_rsa`) **nunca** se comparte, es la que nos identifica.

En GitHub vamos a **Settings > SSH and GPG keys > New SSH key**, le ponemos un título descriptivo (por ejemplo `Gabriel`) y pegamos el contenido en el campo *Key*.

![](/assets/img/ImpAppWeb/2026-09-24-11-10-36-image.png)

### 1.5. Comprobar la conexión

```bash
ssh -T git@github.com
```

Si todo es correcto nos responde con un mensaje de bienvenida con nuestro nombre de usuario.

![](/assets/img/ImpAppWeb/2026-09-24-11-11-19-image.png)

## 2. Crear el repositorio remoto

En GitHub pulsamos en **New repository** y lo configuramos así:

![](/assets/img/ImpAppWeb/2026-09-24-11-13-10-image.png)

## 3. Clonar el repositorio remoto

En la página del repositorio pulsamos en **Code** y seleccionamos la pestaña **SSH** para copiar la URL.

```bash
git clone git@github.com:SysGabrielMO/prueba_gabriel.git
cd prueba_gabriel
```

![](/assets/img/ImpAppWeb/2026-09-24-11-51-43-image.png)

![](/assets/img/ImpAppWeb/2026-09-24-11-52-50-image.png)

Al clonar, Git descarga el repositorio completo y configura automáticamente el remoto con el nombre `origin`.

## 4. Comprobaciones

### 4.1. Comprobar que se usa la URL SSH

```bash
cat .git/config
```

En la sección `[remote "origin"]` debe aparecer la URL en formato SSH:

```
url = git@github.com:SysGabrielMO/prueba_gabriel.git
```

![](/assets/img/ImpAppWeb/2026-09-24-11-53-40-image.png)

También podemos verlo con:

```bash
git remote -v
```

![](/assets/img/ImpAppWeb/2026-09-24-11-53-58-image.png)

### 4.2. Comprobar el fichero README.md

```bash
ls -la
cat README.md
```

Este fichero es el que GitHub muestra en la portada del repositorio, y es donde se pone la descripción del proyecto.

![](/assets/img/ImpAppWeb/2026-09-24-11-54-41-image.png)

## 5. Creación y modificación de archivos

Creamos varios archivos, una carpeta y una subcarpeta, para que se vea mejor graficamente, utilizaré printf:

```bash
mkdir -p documentos/apuntes
printf "<h1>Página de prueba</h1>\n" > index.html
printf "/* Estilos de prueba */\n" > estilos.css
printf "Apuntes de la practica 2\n" > documentos/apuntes/notas.md
```

![](/assets/img/ImpAppWeb/2026-09-24-11-56-53-image.png)

![](/assets/img/ImpAppWeb/2026-09-24-11-58-40-image.png)

Comprobamos el estado y subimos los cambios:

```bash
git status
git add .
git commit -m "Añadidos archivos y carpetas de prueba"
git push origin main
```

![](/assets/img/ImpAppWeb/2026-09-24-11-59-41-image.png)

![](/assets/img/ImpAppWeb/2026-09-24-11-59-57-image.png)

Refrescamos la página del repositorio en GitHub y comprobamos que aparecen los archivos y las carpetas.

![](/assets/img/ImpAppWeb/2026-09-24-12-02-04-image.png)

## 6. Segundo repositorio

Ahora vamos a subir a GitHub el repositorio local que creamos en la **práctica 1**.

### 6.1. Crear el repositorio vacío en GitHub

Creamos un repositorio nuevo llamado `practica1gabriel`, esta vez **sin marcar** la opción del README. Si lo inicializamos con README tendría un commit que nuestro repositorio local no conoce y el push daría error.

![](/assets/img/ImpAppWeb/2026-09-24-12-05-55-image.png)

### 6.2. Enlazar el repositorio local con el remoto

Entramos en la carpeta de la práctica 1 y añadimos el remoto:

```bash
cd ~/Documentos/practicas/ImpAppWeb/gabriel_merencio
git remote add origin git@github.com:SysGabrielMO/practica1gabriel.git
git remote -v
```

![](/assets/img/ImpAppWeb/2026-09-24-12-08-43-image.png)

### 6.3. Subir el repositorio

```bash
git branch -M main
git push -u origin main
```

> **¿Por qué `git branch -M main`?** Nuestro repositorio local se creó con la rama `master`, mientras que GitHub usa `main` por defecto. Este comando renombra la rama para que coincidan. La opción `-u` deja `origin main` como destino por defecto, así en los siguientes envíos basta con `git push`.



![](/assets/img/ImpAppWeb/2026-09-24-12-10-05-image.png)

Al hacer el push normal, me ha dado fallo por eso he tenido que utilizar esos parametros.

Comprobamos en GitHub que aparece todo el historial de commits de la práctica 1.

![](/assets/img/ImpAppWeb/2026-09-24-12-10-54-image.png)

![](/assets/img/ImpAppWeb/2026-09-24-12-12-11-image.png)

## 7. Lenguaje Markdown

Markdown es un lenguaje de marcas ligero que permite dar formato a un texto plano con símbolos. Se usa mucho en los ficheros `README.md` de los repositorios porque GitHub lo interpreta y lo muestra ya formateado.

### 7.1. Elementos utilizados

| Elemento          | Nomenclatura                 |
| ----------------- | ---------------------------- |
| Título principal  | `# Título`                   |
| Subtítulo         | `## Subtítulo`               |
| Negrita           | `**texto**`                  |
| Cursiva           | `*texto*`                    |
| Código en línea   | `` `codigo` ``               |
| Bloque de código  | ` ```lenguaje ` ... ` ``` `  |
| Lista ordenada    | `1. Elemento`                |
| Lista desordenada | `- Elemento`                 |
| Enlace            | `[texto](url)`               |
| Imagen            | `![texto alternativo](ruta)` |
| Tabla             | `\| Columna \| Columna \|`   |

### 7.2. Contenido del README.md

Editamos el README del repositorio con `nano README.md` y escribimos lo siguiente:

```markdown
# Repositorio de prueba 2ASIR

## Descripción del proyecto

Este repositorio ha sido creado para la **práctica 2** del módulo de *Implantación
de Aplicaciones Web*. En él se practica el uso de repositorios remotos en GitHub
mediante SSH y el lenguaje de marcas Markdown. Para clonarlo se utiliza el
comando `git clone`.

## Ejemplo de código

```bash
git clone git@github.com:SysGabrielMO/prueba_gabriel.git
cd prueba_gabriel
git status
```

## Pasos realizados

1. Configuración de la clave SSH en GitHub.
2. Creación del repositorio remoto.
3. Clonado del repositorio en local.
4. Subida de los cambios con push.

## Tecnologías utilizadas

- Git
- GitHub
- Markdown
- Debian GNU/Linux

## Enlaces

- [Mi Github](https://github.com/SysGabrielMO)
- [Apuntes de la práctica](documentos/apuntes/notas.md)

## Logo de Git

![Logo de Git](https://git-scm.com/images/logos/downloads/Git-Logo-2Color.png)

## Comandos básicos

| Comando | Descripción |
| --- | --- |
| `git init` | Inicializa un repositorio local |
| `git add` | Añade cambios al staging area |
| `git commit` | Guarda los cambios en el repositorio |
| `git push` | Envía los commits al repositorio remoto |
| `git pull` | Descarga los cambios del repositorio remoto |
```

Guardamos y subimos los cambios:

```bash
git add README.md
git commit -m "Actualizado README con formato Markdown"
git push
```

![](/assets/img/ImpAppWeb/2026-09-24-12-29-00-image.png)

Comprobamos en GitHub cómo se ve el README ya renderizado:

![](/assets/img/ImpAppWeb/2026-09-24-12-29-22-image.png)



![](/assets/img/ImpAppWeb/2026-09-24-12-29-42-image.png)

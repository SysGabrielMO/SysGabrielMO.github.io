---
title: "Compilación de ripgrep con Rust"
asignatura: ASO
date: 2026-09-26
resumen: "Compilo e instalo ripgrep 15.2.0 desde el código fuente en Debian 13 usando la cadena de herramientas de Rust: rustup para instalarla, cargo build --release para compilar y cargo test para pasar los tests, dejando el binario, el manual y el autocompletado en /opt/ripgrep. Voy comparando cada paso con su equivalente en un proyecto con configure y Makefile, y acabo con una desinstalación limpia comprobando que apt y dpkg nunca controlan el programa."
tags:
  - Rust
  - Cargo
  - Compilación desde fuentes
  - ripgrep
  - Debian
---

---

## 1. Instalación de las herramientas

```bash
gabriel@debian-ansible:~$ sudo apt install build-essential curl
build-essential ya está en su versión más reciente (12.12).
Installing:
  curl

Summary:
  Upgrading: 0, Installing: 1, Removing: 0, Not Upgrading: 0
  Download size: 270 kB
  Space needed: 507 kB / 5.945 MB available

gabriel@debian-ansible:~$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
info: downloading installer

Welcome to Rust!
```

En el menú del instalador seleccionamos la **opción 1**.

Cargamos el entorno y comprobamos las versiones:

```bash
gabriel@debian-ansible:~$ source ~/.cargo/env
gabriel@debian-ansible:~$ rustc --version
rustc 1.98.1 (48a229cea 2026-09-01)
gabriel@debian-ansible:~$ cargo --version
cargo 1.98.1 (797e8a9bc 2026-08-05)
```

---

## 2. Descarga del código fuente

```bash
gabriel@debian-ansible:~$ mkdir -p ~/compilacion
gabriel@debian-ansible:~$ cd compilacion/
gabriel@debian-ansible:~/compilacion$ wget https://github.com/BurntSushi/ripgrep/archive/refs/tags/15.2.0.tar.gz -O ripgrep-15.2.0.tar.gz
--2026-09-26 12:04:05--  https://github.com/BurntSushi/ripgrep/archive/refs/tags/15.2.0.tar.gz
Resolviendo github.com (github.com)... 140.82.121.4
Conectando con github.com (github.com)[140.82.121.4]:443... conectado.
Petición HTTP enviada, esperando respuesta... 302 Found
Localización: https://codeload.github.com/BurntSushi/ripgrep/tar.gz/refs/tags/15.2.0 [siguiendo]
--2026-09-26 12:04:06--  https://codeload.github.com/BurntSushi/ripgrep/tar.gz/refs/tags/15.2.0
Resolviendo codeload.github.com (codeload.github.com)... 140.82.121.10
Conectando con codeload.github.com (codeload.github.com)[140.82.121.10]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: no especificado [application/x-gzip]
Grabando a: «ripgrep-15.2.0.tar.gz»

ripgrep-15.2.0.tar.gz         [     <=>                              ] 592,30K   582KB/s    en 1,0s

2026-09-26 12:04:08 (582 KB/s) - «ripgrep-15.2.0.tar.gz» guardado [606519]

gabriel@debian-ansible:~/compilacion$ ls
ripgrep-15.2.0.tar.gz
gabriel@debian-ansible:~/compilacion$ tar -zxvf ripgrep-15.2.0.tar.gz
```

---

## 3. Comprobación de los ficheros de compilación

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ ls -l
total 276
-rw-rw-r--  1 gabriel gabriel  1878 jul 15 18:00 AI_POLICY.md
drwxrwxr-x  3 gabriel gabriel  4096 jul 15 18:00 benchsuite
-rw-rw-r--  1 gabriel gabriel  2246 jul 15 18:00 build.rs
-rw-rw-r--  1 gabriel gabriel 12745 jul 15 18:00 Cargo.lock
-rw-rw-r--  1 gabriel gabriel  3198 jul 15 18:00 Cargo.toml
-rw-rw-r--  1 gabriel gabriel 89888 jul 15 18:00 CHANGELOG.md
drwxrwxr-x  2 gabriel gabriel  4096 jul 15 18:00 ci
-rw-rw-r--  1 gabriel gabriel   213 jul 15 18:00 CONTRIBUTING.md
-rw-rw-r--  1 gabriel gabriel   126 jul 15 18:00 COPYING
drwxrwxr-x 12 gabriel gabriel  4096 jul 15 18:00 crates
-rw-rw-r--  1 gabriel gabriel 42243 jul 15 18:00 FAQ.md
drwxrwxr-x  3 gabriel gabriel  4096 jul 15 18:00 fuzz
-rw-rw-r--  1 gabriel gabriel 40895 jul 15 18:00 GUIDE.md
lrwxrwxrwx  1 gabriel gabriel     8 jul 15 18:00 HomebrewFormula -> pkg/brew
-rw-rw-r--  1 gabriel gabriel  1081 jul 15 18:00 LICENSE-MIT
drwxrwxr-x  4 gabriel gabriel  4096 jul 15 18:00 pkg
-rw-rw-r--  1 gabriel gabriel 21615 jul 15 18:00 README.md
-rw-rw-r--  1 gabriel gabriel  2878 jul 15 18:00 RELEASE-CHECKLIST.md
-rw-rw-r--  1 gabriel gabriel    61 jul 15 18:00 rustfmt.toml
drwxrwxr-x  2 gabriel gabriel  4096 jul 15 18:00 scripts
drwxrwxr-x  3 gabriel gabriel  4096 jul 15 18:00 tests
-rw-rw-r--  1 gabriel gabriel  1211 jul 15 18:00 UNLICENSE
```

Como podemos ver, no existe ningún `configure` ni ningún `Makefile`. En su lugar se encuentran los ficheros `Cargo.toml`, `Cargo.lock` y `build.rs`.

---

## 4. Configuración

En Rust no hay un paso separado de configuración. Para comprobar que nuestra versión de Rust cumple el mínimo exigido, comparamos estas dos salidas:

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ rustc --version
rustc 1.98.1 (48a229cea 2026-09-01)
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ grep rust-version Cargo.toml
rust-version = "1.85"
```

Todo correcto: tenemos **Rust 1.98.1** y ripgrep 15.2.0 necesita como mínimo la versión **1.85**.

---

## 5. Compilación

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ cargo build --release --locked
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ ls -lh target/release/rg
-rwxrwxr-x 2 gabriel gabriel 30M sep 26 12:09 target/release/rg
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ ./target/release/rg --version
ripgrep 15.2.0

features:-pcre2
simd(compile):+SSE2,-SSSE3,-AVX2
simd(runtime):+SSE2,+SSSE3,+AVX2

PCRE2 is not available in this build of ripgrep
```

---

## 6. Ejecución de los tests

Este paso equivale a `make check`:

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ cargo test --release --locked
test result: ok. 332 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 2.02s
```

---

## 7. Instalación

Generamos la página de manual y el autocompletado:

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ ./target/release/rg --generate man > rg.1
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ ./target/release/rg --generate complete-bash > rg.bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ ls -l rg.1 rg.bash
-rw-rw-r-- 1 gabriel gabriel 86726 sep 26 12:27 rg.1
-rw-rw-r-- 1 gabriel gabriel 21362 sep 26 12:27 rg.bash
```

A continuación, instalamos los ficheros:

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ sudo install -Dsm755 target/release/rg /opt/ripgrep/bin/rg
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ sudo install -Dm644 rg.1 /opt/ripgrep/share/man/man1/rg.1
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ sudo install -Dm644 rg.bash /opt/ripgrep/share/bash-completion/completions/rg
```

Opciones de `install`:

| Opción              | Significado                                   |
| ------------------- | --------------------------------------------- |
| `-D`                | Crea los directorios de destino si no existen |
| `-s`                | Pasa `strip` al binario al copiarlo           |
| `-m 755` / `-m 644` | Permisos del fichero instalado                |

---

## 8. Ficheros instalados

Comprobamos los ficheros instalados, el tamaño del binario y sus dependencias:

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ find /opt/ripgrep -type f
/opt/ripgrep/share/man/man1/rg.1
/opt/ripgrep/share/bash-completion/completions/rg
/opt/ripgrep/bin/rg
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ ls -lh /opt/ripgrep/bin/rg
-rwxr-xr-x 1 root root 5,1M sep 26 12:28 /opt/ripgrep/bin/rg
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ ldd /opt/ripgrep/bin/rg
	linux-vdso.so.1 (0x00007f9b3fa34000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f9b3f9fd000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f9b3f20c000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f9b3fa36000)
```

---

## 9. Comprobación del funcionamiento

### Ejecución del binario

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ /opt/ripgrep/bin/rg --version
ripgrep 15.2.0

features:-pcre2
simd(compile):+SSE2,-SSSE3,-AVX2
simd(runtime):+SSE2,+SSSE3,+AVX2

PCRE2 is not available in this build of ripgrep.
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ /opt/ripgrep/bin/rg -n "fn main" ~/compilacion/ripgrep-15.2.0/crates/core/
/home/gabriel/compilacion/ripgrep-15.2.0/crates/core/main.rs
43:fn main() -> ExitCode {
```

### Comprobación de que no interfiere con el sistema de paquetes

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ apt policy ripgrep
ripgrep:
  Instalados: (ninguno)
  Candidato:  14.1.1-1+b4
  Tabla de versión:
     14.1.1-1+b4 500
        500 http://deb.debian.org/debian trixie/main amd64 Packages
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ dpkg -S /opt/ripgrep/bin/rg
dpkg-query: no se ha encontrado ningún paquete que corresponda con el patrón /opt/ripgrep/bin/rg.
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ dpkg -S /usr/bin/ls
coreutils: /usr/bin/ls
```

---

## 10. Desinstalación limpia

### Paso 1: borrar la instalación

Cargo no tiene un equivalente a `make uninstall` para ficheros copiados con `install`. Como todo está en su propio prefijo, basta con borrar el directorio:

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ sudo rm -rf /opt/ripgrep
```

### Paso 2: comprobar que no queda nada

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ ls /opt
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ which rg || printf "rg ya no está instalado\n"
rg ya no está instalado
```

### Paso 3: limpiar las fuentes (opcional)

```bash
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ cargo clean
     Removed 448 files, 323.9MiB total
gabriel@debian-ansible:~/compilacion/ripgrep-15.2.0$ cd
gabriel@debian-ansible:~$ pwd
/home/gabriel
gabriel@debian-ansible:~$ rm -rf ~/compilacion
```

`cargo clean` borra la carpeta `target/`, que con los tests incluidos ocupa bastante. Es el equivalente a `make clean`. En Rust no hace falta un `distclean`, porque no hay ficheros generados por `configure`.

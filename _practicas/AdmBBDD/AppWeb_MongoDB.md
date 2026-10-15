---
title: "Aplicación web con MongoDB e inyección NoSQL"
asignatura: AdmBBDD
date: 2026-10-15
resumen: "Hago una aplicación web con Flask y PyMongo, en un contenedor LXC, que se autentica contra un MongoDB remoto y muestra solo las colecciones que permiten los roles de cada usuario. Después demuestro una inyección NoSQL con operadores como $ne y $gt en una consulta pública de clientes, la corrijo validando la entrada, desactivo JavaScript en el servidor y compruebo que el ataque ya no funciona."
tags:
  - MongoDB
  - NoSQL
  - Flask
  - Python
  - LXC
  - Seguridad
---

## Enunciado

**Parte obligatoria:** realizar una aplicación web, en cualquier lenguaje, que conecte con el servidor MongoDB desde un cliente remoto tras autenticarse, muestre las colecciones accesibles para ese usuario y permita acceder a los documentos de alguna de ellas.

**Parte opcional:** comprobar que la aplicación web es vulnerable a ataques de inyección NoSQL, explicar las medidas que hay que tomar para que deje de serlo y demostrar que, después de aplicarlas, ese tipo de ataques ya no es posible.

---

## Entorno

| Elemento                    | Valor                                                                      |
| --------------------------- | -------------------------------------------------------------------------- |
| Servidor MongoDB            | VM KVM `srv-mongodb` (Debian 13), IP `192.168.122.93`                      |
| Versión de MongoDB          | 9.0.2, con autenticación activada y escuchando en `0.0.0.0`                |
| Contenedor de la aplicación | LXC `app-web-mongo` (Debian 13), IP fija `192.168.122.50`                  |
| Equipo anfitrión            | Portátil `debian-gabriel` (Debian 13), red `default` de libvirt (`virbr0`) |
| Lenguaje y librerías        | Python 3.13, Flask 3.1.1, PyMongo 4.10.1                                   |
| Servidor de aplicaciones    | Gunicorn 23.0.0, gestionado por systemd                                    |
| URL de la aplicación        | `http://192.168.122.50:5000`                                               |

La instalación de MongoDB y su configuración para el acceso remoto están en el documento *Instalación de MongoDB 9.0 en Debian 13 con acceso remoto*.

## Arquitectura

```
                    Red default de libvirt (192.168.122.0/24)
 ┌──────────────────────┐                       ┌───────────────────────┐
 │ Contenedor LXC       │      TCP 27017        │ VM srv-mongodb        │
 │ app-web-mongo        │ ────────────────────► │ MongoDB 9.0.2         │
 │ 192.168.122.50:5000  │  mongodb://...        │ 192.168.122.93        │
 │ Flask + Gunicorn     │                       │ BD "tienda"           │
 └──────────▲───────────┘                       └───────────────────────┘
            │ HTTP 5000
 ┌──────────┴───────────┐
 │ Navegador / curl     │
 │ (portátil)           │
 └──────────────────────┘
```

La aplicación tiene **dos partes**:

| Parte                                             | URL                                                | Con qué usuario de MongoDB se conecta                                                                                                                     |
| ------------------------------------------------- | -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Administración** (parte obligatoria)            | `/`, `/colecciones`, `/coleccion/<bd>/<colección>` | Con el usuario que se escribe en el login (`admin`, `gabriel`, `invitado`...). Cada uno ve solo lo que le permiten sus roles en MongoDB.                  |
| **Consulta pública de clientes** (parte opcional) | `/consulta`                                        | Siempre con la **cuenta de servicio `portal`**, que tiene el mínimo privilegio. El visitante no se autentica en MongoDB: solo escribe su nombre y su PIN. |

Es como funciona una web real: la zona de administración usa los usuarios de la base de datos, y la parte pública usa una sola cuenta de servicio, mientras que el control de quién ve qué lo hace el código de la aplicación.

---

## 1. Preparación de MongoDB

Todos los comandos de este apartado se ejecutan en `srv-mongodb`, conectado como administrador:

```bash
mongosh "mongodb://127.0.0.1:27017/?authSource=admin" -u admin -p
```

### 1.1. Datos de prueba

Se crea la base de datos `tienda` con tres colecciones. En MongoDB, la base de datos y las colecciones se crean automáticamente al insertar el primer documento.

```javascript
use tienda
```

```javascript
db.productos.insertMany([
  { nombre: "Teclado mecánico", precio: 59.9, stock: 12, categoria: "periféricos" },
  { nombre: "Ratón inalámbrico", precio: 24.5, stock: 30, categoria: "periféricos" },
  { nombre: "Monitor 27 pulgadas", precio: 189, stock: 5, categoria: "pantallas" }
])
```

```javascript
db.clientes.insertMany([
  { nombre: "Ana López", ciudad: "Sevilla", pin: 1234, telefono: "600111222", direccion: "Calle Sierpes 1" },
  { nombre: "Luis Pérez", ciudad: "Dos Hermanas", pin: 5678, telefono: "600333444", direccion: "Av. de España 10" },
  { nombre: "Marta Ruiz", ciudad: "Utrera", pin: 4321, telefono: "600555666", direccion: "Plaza del Altozano 3" }
])
```

```javascript
db.pedidos.insertMany([
  { cliente: "Ana López", productos: ["Teclado mecánico"], total: 59.9, fecha: new Date() },
  { cliente: "Luis Pérez", productos: ["Ratón inalámbrico", "Monitor 27 pulgadas"], total: 213.5, fecha: new Date() }
])
```

Los clientes tienen un **PIN**, que es el dato privado que se usará en la parte de la inyección NoSQL.

![Colecciones de la base de datos tienda y documentos de clientes](/assets/img/AdmBBDD/collections_tienda.png)

### 1.2. Roles personalizados

Los roles de MongoDB permiten dar permisos a nivel de colección. Se crean en la base de datos `admin`:

```javascript
use admin
```

```javascript
db.createRole({
  role: "lectorProductos",
  privileges: [ { resource: { db: "tienda", collection: "productos" }, actions: ["find"] } ],
  roles: []
})
```

```javascript
db.createRole({
  role: "portalClientes",
  privileges: [ { resource: { db: "tienda", collection: "clientes" }, actions: ["find"] } ],
  roles: []
})
```

| Rol               | Permiso                                            |
| ----------------- | -------------------------------------------------- |
| `lectorProductos` | Solo leer (`find`) la colección `tienda.productos` |
| `portalClientes`  | Solo leer (`find`) la colección `tienda.clientes`  |

![Roles personalizados con sus privilegios](/assets/img/AdmBBDD/roles.png)

### 1.3. Usuarios

Las contraseñas se piden por teclado con `passwordPrompt()`, para que no queden escritas ni en el historial ni en las capturas.

```javascript
db.createUser({ user: "gabriel", pwd: passwordPrompt(), roles: [ { role: "read", db: "tienda" } ] })
```

```javascript
db.createUser({ user: "invitado", pwd: passwordPrompt(), roles: [ { role: "lectorProductos", db: "admin" } ] })
```

```javascript
db.createUser({ user: "portal", pwd: passwordPrompt(), roles: [ { role: "portalClientes", db: "admin" } ] })
```

| Usuario    | Rol                   | Qué puede hacer                                                              |
| ---------- | --------------------- | ---------------------------------------------------------------------------- |
| `admin`    | `root`                | Todo (creado durante la instalación)                                         |
| `gabriel`  | `read` sobre `tienda` | Leer las tres colecciones de `tienda`, sin poder modificarlas                |
| `invitado` | `lectorProductos`     | Leer solo `tienda.productos`                                                 |
| `portal`   | `portalClientes`      | Leer solo `tienda.clientes`. Es la cuenta de servicio de la consulta pública |

![Usuarios creados y sus roles](/assets/img/AdmBBDD/usuarios.png)

### 1.4. Comprobación de los permisos

Antes de hacer la aplicación, se comprueba desde el portátil con `mongosh` que cada usuario solo puede acceder a lo que le corresponde.

**`invitado`:** puede leer `productos`, pero no `clientes`.

![Permisos del usuario invitado](/assets/img/AdmBBDD/invitado_comprobacion.png)

**`gabriel`:** puede ver y leer las tres colecciones de `tienda`.

![Permisos del usuario gabriel](/assets/img/AdmBBDD/gabriel_comprobacion.png)

**`portal`:** puede leer `clientes`, pero no `productos`.

![Permisos del usuario portal](/assets/img/AdmBBDD/portal_Comprobacion.png)

> **Nota sobre `show collections`:** con `invitado` y `portal`, este comando da `not authorized`. No es un fallo de permisos: `mongosh` lo ejecuta con `authorizedCollections: false`, es decir, pide la lista de **todas** las colecciones, y para eso hace falta un permiso sobre toda la base de datos que estos usuarios no tienen a propósito. La aplicación web usa `authorizedCollections: True` ("dame solo las colecciones que puedo ver"), así que a ella no le pasa.

---

## 2. Contenedor LXC para la aplicación

La aplicación se ejecuta en un contenedor LXC del portátil, conectado al bridge `virbr0` de libvirt. Así está en la misma red que `srv-mongodb` y actúa como un **cliente remoto** más.

Se conecta directamente al bridge porque las reglas de cortafuegos que crea libvirt para la red `default` solo admiten, desde fuera de `192.168.122.0/24`, el tráfico de conexiones ya establecidas. Un contenedor en otra red no podría abrir conexiones nuevas hacia la VM.

Todos los comandos de este apartado se ejecutan en el **portátil**.

### 2.1. Instalación de LXC y creación del contenedor

```bash
sudo apt update
sudo apt install -y lxc
sudo lxc-create -n app-web-mongo -t download -- -d debian -r trixie -a amd64
```

### 2.2. Configuración de red

En `/var/lib/lxc/app-web-mongo/config` se sustituye la línea `lxc.net.0.type = empty` (contenedor sin red) por:

```
lxc.net.0.type = veth
lxc.net.0.link = virbr0
lxc.net.0.flags = up
lxc.net.0.hwaddr = 00:16:3e:aa:bb:01
lxc.start.auto = 1
```

| Línea                | Función                                                                                                 |
| -------------------- | ------------------------------------------------------------------------------------------------------- |
| `veth` + `virbr0`    | Crea una interfaz virtual conectada al bridge de libvirt                                                |
| `hwaddr`             | MAC fija, para poder reservar siempre la misma IP. El prefijo `00:16:3e` es el habitual en contenedores |
| `lxc.start.auto = 1` | El contenedor arranca automáticamente con el sistema                                                    |

![Configuración de red del contenedor](/assets/img/AdmBBDD/f2-01-config-red.png)

### 2.3. IP fija mediante reserva DHCP

Se reserva la IP `192.168.122.50` para la MAC del contenedor en el DHCP de libvirt:

```bash
sudo virsh net-update default add ip-dhcp-host \
  "<host mac='00:16:3e:aa:bb:01' name='app-web-mongo' ip='192.168.122.50'/>" --live --config
```

`--live` aplica el cambio en ese momento y `--config` lo mantiene después de reiniciar.

### 2.4. Arranque

```bash
sudo lxc-start -n app-web-mongo
sudo lxc-ls -f
```

![Contenedor en ejecución con la IP reservada](/assets/img/AdmBBDD/resumen_contenedor.png)

### 2.5. Comprobación de la conectividad

Se comprueba que el contenedor tiene salida a Internet (para instalar paquetes) y que llega al puerto de MongoDB. Como la imagen mínima todavía no tiene Python, la prueba de MongoDB se hace con `bash`:

```bash
sudo lxc-attach -n app-web-mongo -- apt update
sudo lxc-attach -n app-web-mongo -- bash -c 'timeout 3 bash -c "</dev/tcp/192.168.122.93/27017" && printf "MongoDB accesible\n"'
```

![Conectividad del contenedor con Internet y con MongoDB](/assets/img/AdmBBDD/conexion-y-accede-mongo.png)

---

## 3. Aplicación web (parte obligatoria)

### 3.1. Instalación de dependencias

Dentro del contenedor (`sudo lxc-attach -n app-web-mongo`):

```bash
apt install -y python3-flask python3-pymongo gunicorn nano
useradd --system --no-create-home appuser
mkdir -p /opt/mongo-web/templates
```

Se usan los paquetes de Debian, así que no hace falta `pip` ni un entorno virtual. El usuario `appuser` sirve para que la aplicación no se ejecute como root.

![Instalación de las dependencias](/assets/img/AdmBBDD/apt-install.png)

### 3.2. Estructura de la aplicación

```
/opt/mongo-web/
├── app.py                  Lógica de la aplicación
└── templates/
    ├── base.html           Cabecera y estilos comunes
    ├── login.html          Pantalla de autenticación
    ├── colecciones.html    Colecciones accesibles y cadena de conexión
    ├── documentos.html     Documentos de una colección
    └── consulta.html       Consulta pública de clientes (parte opcional)
```

El código completo está en el [Anexo](#anexo-código-completo).

### 3.3. Funcionamiento

**Cadena de conexión.** La aplicación construye la cadena de conexión de MongoDB con los datos del formulario:

```python
def construir_cadena(servidor, puerto, usuario, contrasena, auth_source):
    return "mongodb://" + quote_plus(usuario) + ":" + quote_plus(contrasena) + "@" + servidor + ":" + str(puerto) + "/?authSource=" + quote_plus(auth_source)
```

Resultado: `mongodb://gabriel:CONTRASEÑA@192.168.122.93:27017/?authSource=admin`

- `quote_plus()` codifica el usuario y la contraseña. Sin esto, una contraseña con caracteres como `@`, `:` o `/` rompería la cadena, porque tienen un significado especial en una URL.
- `authSource=admin` indica que los usuarios están definidos en la base de datos `admin`.

Como referencia, cada gestor tiene su propio formato de cadena de conexión:

| Gestor     | Formato                                                     |
| ---------- | ----------------------------------------------------------- |
| MongoDB    | `mongodb://usuario:contraseña@host:27017/?authSource=admin` |
| PostgreSQL | `postgresql://usuario:contraseña@host:5432/basedatos`       |
| MySQL      | `mysql://usuario:contraseña@host:3306/basedatos`            |
| Oracle     | `usuario/contraseña@host:1521/servicio`                     |

**Autenticación.** Con la cadena se crea la conexión y se lanza un `ping`. Si el usuario o la contraseña son incorrectos, MongoDB rechaza la autenticación y la aplicación muestra el error.

```python
def conectar(cadena):
    cliente = MongoClient(cadena, serverSelectionTimeoutMS=3000)
    cliente.admin.command("ping")
    return cliente
```

**Colecciones según los permisos.** Esta función cumple el requisito principal del enunciado. Con `authorizedDatabases=True` y `authorizedCollections=True`, MongoDB devuelve **solo** las bases de datos y colecciones a las que el usuario tiene permiso:

```python
def colecciones_accesibles(cliente):
    resultado = {}
    bases = [bd["name"] for bd in cliente.list_databases(authorizedDatabases=True, nameOnly=True)]
    for nombre_bd in bases:
        try:
            colecciones = cliente[nombre_bd].list_collection_names(authorizedCollections=True)
        except PyMongoError:
            colecciones = []
        if len(colecciones) > 0:
            resultado[nombre_bd] = sorted(colecciones)
    return resultado
```

**Sesiones.** La contraseña no se guarda en el navegador. La conexión abierta se queda en el servidor, en un diccionario, y el navegador solo guarda un token aleatorio (`secrets.token_hex(16)`) que la identifica. Al cerrar la sesión, la conexión se cierra.

### 3.4. Servicio systemd

La aplicación se ejecuta con Gunicorn como servicio de systemd. Así arranca sola con el contenedor y se reinicia si falla. Fichero `/etc/systemd/system/mongo-web.service`:

```ini
[Unit]
Description=Cliente web de MongoDB (Flask + Gunicorn)
After=network-online.target
Wants=network-online.target

[Service]
User=appuser
Group=appuser
WorkingDirectory=/opt/mongo-web
Environment=MONGO_SERVIDOR=192.168.122.93
Environment=MONGO_PUERTO=27017
Environment=MODO_SEGURO=1
Environment=PORTAL_USUARIO=portal
Environment=PORTAL_CONTRASENA=********
ExecStart=/usr/bin/gunicorn --workers 1 --threads 4 --bind 0.0.0.0:5000 app:app
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

- Se usa `--workers 1` porque las conexiones se guardan en memoria. Con varios procesos, cada uno tendría su propio diccionario y la sesión se perdería entre peticiones.
- Como el fichero contiene la contraseña de `portal`, solo puede leerlo root (`chmod 600`).

```bash
systemctl daemon-reload
systemctl enable --now mongo-web
```

![Servicio de la aplicación en ejecución](/assets/img/AdmBBDD/serv-active.png)

### 3.5. Pruebas

**Pantalla de autenticación**

![Pantalla de autenticación](/assets/img/AdmBBDD/menu_final.png)

**Credenciales incorrectas:** MongoDB rechaza la autenticación (`AuthenticationFailed`) y la aplicación no deja pasar.

![Login con contraseña incorrecta](/assets/img/AdmBBDD/login_contrasena_incorrecta.png)

**Acceso según los permisos.** Cada usuario ve solo lo que le permiten sus roles. En cada pantalla aparece la cadena de conexión utilizada, con la contraseña oculta.

| Usuario    | Colecciones que ve                                              |
| ---------- | --------------------------------------------------------------- |
| `gabriel`  | `tienda`: `clientes`, `pedidos` y `productos`                   |
| `invitado` | `tienda`: solo `productos`                                      |
| `admin`    | Todas las bases de datos: `admin`, `config`, `local` y `tienda` |

![Colecciones accesibles para gabriel](/assets/img/AdmBBDD/gabriel_entrar.png)

![Colecciones accesibles para invitado](/assets/img/AdmBBDD/invitado_entrar.png)

![Colecciones accesibles para admin](/assets/img/AdmBBDD/admin_entrar.png)

**Acceso a los documentos de una colección**

![Documentos de tienda.productos](/assets/img/AdmBBDD/documentos.tienda.producto.png)

---

## 4. Inyección NoSQL (parte opcional)

### 4.1. Qué es la inyección NoSQL

En MongoDB, las consultas son objetos que pueden contener **operadores**, como `$ne` ("distinto de") o `$gt` ("mayor que"). La inyección NoSQL consiste en colar uno de estos operadores en un dato que la aplicación usa para construir una consulta. Así se cambia el significado de la consulta y se obtiene información que no se debería ver.

### 4.2. Por qué la pantalla de autenticación no es vulnerable

En la pantalla de login **no se construye ninguna consulta**. El usuario y la contraseña se envían directamente a MongoDB, que los comprueba con su propio mecanismo de autenticación (SCRAM). Si se escribe un operador como `{"$ne": ""}` en la contraseña, MongoDB lo trata como una contraseña normal y la rechaza (ver la captura del apartado 3.5).

Por eso, la parte vulnerable está en la **consulta pública de clientes**, que sí construye una consulta con los datos del usuario.

### 4.3. Escenario: consulta pública de clientes

La página `/consulta` simula la web de una tienda donde un cliente escribe su **nombre** y su **PIN** para ver **sus** datos. Por debajo, la aplicación ejecuta con la cuenta de servicio `portal`:

```python
db.clientes.find({"nombre": <nombre escrito>, "pin": <PIN escrito>})
```

La página muestra también el **filtro que se envía a MongoDB**. Una aplicación real no lo enseñaría, pero aquí sirve para ver qué ocurre en cada prueba.

### 4.4. El código vulnerable

```python
def convertir_valor(texto):
    try:
        valor = json.loads(texto)
    except ValueError:
        valor = texto
    return valor


def construir_filtro_vulnerable(nombre, pin):
    return {"nombre": convertir_valor(nombre), "pin": convertir_valor(pin)}
```

El programador usa `json.loads()` para que el PIN `1234` llegue a MongoDB como número y no como texto. El problema es que `json.loads()` convierte **cualquier JSON** en una estructura de Python. Si el usuario escribe `{"$ne": 0}`, se convierte en un diccionario y MongoDB lo interpreta como un **operador**.

> En Flask, `request.form.get()` siempre devuelve texto, así que una aplicación Flask no es vulnerable por defecto. La vulnerabilidad aparece cuando el programador convierte la entrada del usuario en estructuras, por ejemplo con `json.loads()` o pasando `request.get_json()` directamente a un `find()`. Es un error habitual en aplicaciones reales, sobre todo en APIs que reciben JSON.

### 4.5. Demostración del ataque

Modo vulnerable activado en el servicio:

![Modo vulnerable activado](/assets/img/AdmBBDD/f4-01-modo-vulnerable.png)

Todas las pruebas se hacen **sin iniciar sesión**: la barra superior no muestra ningún usuario. El atacante no necesita ninguna cuenta.

| #   | Nombre        | PIN          | Filtro enviado a MongoDB                     | Resultado                              |
| --- | ------------- | ------------ | -------------------------------------------- | -------------------------------------- |
| 1   | `Ana López`   | `1234`       | `{"nombre": "Ana López", "pin": 1234}`       | 1 cliente (uso normal)                 |
| 2   | `Ana López`   | `0000`       | `{"nombre": "Ana López", "pin": "0000"}`     | 0 clientes (PIN incorrecto)            |
| 3   | `Ana López`   | `{"$ne": 0}` | `{"nombre": "Ana López", "pin": {"$ne": 0}}` | ⚠️ **Datos de Ana sin conocer su PIN** |
| 4   | `{"$ne": ""}` | `{"$gt": 0}` | `{"nombre": {"$ne": ""}, "pin": {"$gt": 0}}` | ⚠️ **Todos los clientes**              |

**Prueba 1 — Uso normal**

![Consulta normal con el PIN correcto](/assets/img/AdmBBDD/f4-02-normal.png)

**Prueba 2 — PIN incorrecto**

![Consulta con PIN incorrecto](/assets/img/AdmBBDD/f4-03-pin-incorrecto.png)

**Prueba 3 — Acceso a los datos de un cliente sin conocer su PIN.** El filtro pide "el cliente Ana López con un PIN **distinto de 0**". Como el PIN real (1234) es distinto de 0, la condición se cumple y se muestran los datos privados de Ana.

![Inyección en el campo PIN](/assets/img/AdmBBDD/f4-04-ataque-pin.png)

**Prueba 4 — Extracción de todos los clientes.** El filtro pide "clientes con nombre distinto de vacío y PIN mayor que 0", que cumplen todos.

![Inyección en los dos campos](/assets/img/AdmBBDD/f4-05-ataque-todos.png)

**Ataque sin formulario.** Un atacante real no usa el formulario, sino peticiones directas. Con `curl` se obtienen los teléfonos de todos los clientes:

```bash
curl -s -X POST http://192.168.122.50:5000/consulta \
  --data-urlencode 'nombre={"$ne": ""}' \
  --data-urlencode 'pin={"$gt": 0}' | grep -E "Resultados|telefono"
```

![Ataque con curl en modo vulnerable](/assets/img/AdmBBDD/f4-06-curl-vulnerable.png)

Los `&#34;` de la salida son comillas dobles: Flask las codifica al generar el HTML.

---

## 5. Medidas para evitar la inyección NoSQL

### 5.1. Medida 1 — Validar la entrada en el código de la aplicación

Es la medida principal y la que corrige la vulnerabilidad:

```python
def construir_filtro_seguro(nombre, pin):
    filtro = None
    nombre_valido = isinstance(nombre, str) and len(nombre) > 0 and len(nombre) <= 100
    pin_valido = isinstance(pin, str) and len(pin) == 4 and pin.isascii() and pin.isdigit()
    if nombre_valido and pin_valido:
        filtro = {"nombre": nombre, "pin": int(pin)}
    return filtro
```

| Antes (vulnerable)                                       | Después (seguro)                                                                           |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `json.loads()` sobre la entrada del usuario              | La entrada se usa siempre como **texto**                                                   |
| Sin validación                                           | El PIN debe tener exactamente 4 cifras (`isdigit()`) y el nombre, entre 1 y 100 caracteres |
| El PIN puede ser cualquier estructura                    | El PIN se convierte explícitamente con `int()`                                             |
| El usuario puede introducir operadores (`$ne`, `$gt`...) | Los operadores se tratan como texto normal                                                 |
| —                                                        | Si los datos no son válidos, no se hace ninguna consulta                                   |

Además, el programa decide qué campos tiene el filtro (`"nombre"` y `"pin"` están escritos en el código): el usuario solo puede aportar valores.

![Código de las dos versiones del filtro](/assets/img/AdmBBDD/f4-14-codigo-filtros.png)

### 5.2. Medida 2 — Desactivar JavaScript en el servidor MongoDB

MongoDB permite ejecutar JavaScript en el servidor con el operador `$where`. Es la forma más peligrosa de inyección NoSQL, porque permite ejecutar código. Si la aplicación no lo necesita, se desactiva en `/etc/mongod.conf` de `srv-mongodb`:

```yaml
security:
  authorization: enabled
  javascriptEnabled: false
```

```bash
sudo systemctl restart mongod
```

![JavaScript desactivado en MongoDB](/assets/img/AdmBBDD/f4-07-mongod-conf.png)

### 5.3. Medida 3 — Mínimo privilegio

La cuenta de servicio `portal` solo tiene permiso `find` sobre `tienda.clientes` (rol `portalClientes`, apartado 1.2). Aunque un atacante consiguiera inyectar una consulta, no podría leer otras colecciones ni modificar o borrar datos. Esta medida no evita la inyección, pero **limita sus daños**.

### 5.4. Activación de la versión segura

La aplicación incluye las dos versiones del filtro y usa una u otra según la variable `MODO_SEGURO` del servicio. Así se puede comparar el comportamiento antes y después con las mismas pruebas.

```bash
sudo lxc-attach -n app-web-mongo -- sed -i 's/MODO_SEGURO=0/MODO_SEGURO=1/' /etc/systemd/system/mongo-web.service
sudo lxc-attach -n app-web-mongo -- systemctl daemon-reload
sudo lxc-attach -n app-web-mongo -- systemctl restart mongo-web
```

![Modo seguro activado](/assets/img/AdmBBDD/f4-08-modo-seguro.png)

---

## 6. Demostración de que el ataque ya no es posible

Se repiten las mismas pruebas, también sin iniciar sesión:

| #   | Nombre        | PIN          | Resultado                                         |
| --- | ------------- | ------------ | ------------------------------------------------- |
| 1   | `Ana López`   | `1234`       | 1 cliente: **el uso normal sigue funcionando**    |
| 3   | `Ana López`   | `{"$ne": 0}` | ✅ "Datos no válidos": no se hace ninguna consulta |
| 4   | `{"$ne": ""}` | `{"$gt": 0}` | ✅ "Datos no válidos": no se hace ninguna consulta |
| 5   | `{"$ne": ""}` | `1234`       | ✅ 0 clientes: el operador se trata como texto     |

**Prueba 1 — Uso normal**

![Consulta normal en modo seguro](/assets/img/AdmBBDD/f4-09-seguro-normal.png)

**Prueba 3 — Inyección en el PIN.** El PIN no tiene 4 cifras, así que la aplicación rechaza los datos antes de consultar MongoDB. Por eso ya no aparece el recuadro del filtro.

![Inyección en el PIN rechazada](/assets/img/AdmBBDD/f4-10-seguro-ataque-pin.png)

**Prueba 4 — Inyección en los dos campos**

![Inyección en los dos campos rechazada](/assets/img/AdmBBDD/f4-11-seguro-ataque-todos.png)

**Prueba 5 — Operador en el nombre con un PIN válido.** Es la prueba más clara. El filtro enviado es:

```
{"nombre": "{\"$ne\": \"\"}", "pin": 1234}
```

El operador aparece **entre comillas, como texto**: MongoDB busca un cliente que se llame literalmente `{"$ne": ""}`, que no existe. El operador ha perdido su significado especial.

![Operador convertido en texto](/assets/img/AdmBBDD/f4-12-seguro-texto.png)

**Ataque con `curl`.** El mismo ataque que antes devolvía los teléfonos de todos los clientes ahora solo obtiene el mensaje de error:

```bash
curl -s -X POST http://192.168.122.50:5000/consulta \
  --data-urlencode 'nombre={"$ne": ""}' \
  --data-urlencode 'pin={"$gt": 0}' | grep -E "Resultados|telefono|Datos no" | sed 's/&#34;/"/g'
```

![Ataque con curl en modo seguro](/assets/img/AdmBBDD/f4-13-curl-seguro.png)

---

## 7. Problemas encontrados

| Problema                                                    | Causa                                                                                                                                  | Solución                                                                                      |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Error 500 al entrar en `/colecciones` con cualquier usuario | En PyMongo 4.10 (la versión de Debian 13), `list_database_names()` no acepta el parámetro `authorizedDatabases` y lanza un `TypeError` | Usar `list_databases(authorizedDatabases=True, nameOnly=True)` y quedarse con el campo `name` |
| La cadena mostrada salía como `gabriel:%2A%2A%2A%2A@...`    | `quote_plus()` también codificaba los asteriscos que ocultan la contraseña                                                             | Construir la cadena que se muestra sin pasar los asteriscos por `quote_plus()`                |
| Los cambios en las plantillas no se veían en el navegador   | Fuera del modo de desarrollo, Flask guarda las plantillas en memoria y no vuelve a leerlas del disco                                   | Reiniciar el servicio después de cada cambio: `systemctl restart mongo-web`                   |
| `show collections` da `not authorized` con `invitado`       | `mongosh` pide todas las colecciones (`authorizedCollections: false`)                                                                  | No es un fallo: la aplicación usa `authorizedCollections: True`                               |

---

## Anexo: código completo

### `app.py`

```python
import json
import os
import secrets
from flask import Flask, render_template, request, redirect, url_for, session
from pymongo import MongoClient
from pymongo.errors import PyMongoError
from bson import json_util
from urllib.parse import quote_plus

app = Flask(__name__)
app.secret_key = secrets.token_hex(32)

SERVIDOR_POR_DEFECTO = os.environ.get("MONGO_SERVIDOR", "192.168.122.93")
PUERTO_POR_DEFECTO = int(os.environ.get("MONGO_PUERTO", "27017"))
LIMITE_DOCUMENTOS = 50

clientes = {}


def construir_cadena(servidor, puerto, usuario, contrasena, auth_source):
    return "mongodb://" + quote_plus(usuario) + ":" + quote_plus(contrasena) + "@" + servidor + ":" + str(puerto) + "/?authSource=" + quote_plus(auth_source)


def conectar(cadena):
    cliente = MongoClient(cadena, serverSelectionTimeoutMS=3000)
    cliente.admin.command("ping")
    return cliente


def obtener_cliente():
    token = session.get("token")
    cliente = None
    if token is not None:
        cliente = clientes.get(token)
    return cliente


def colecciones_accesibles(cliente):
    resultado = {}
    bases = [bd["name"] for bd in cliente.list_databases(authorizedDatabases=True, nameOnly=True)]
    for nombre_bd in bases:
        try:
            colecciones = cliente[nombre_bd].list_collection_names(authorizedCollections=True)
        except PyMongoError:
            colecciones = []
        if len(colecciones) > 0:
            resultado[nombre_bd] = sorted(colecciones)
    return resultado


def leer_documentos(cliente, nombre_bd, nombre_coleccion, limite):
    cursor = cliente[nombre_bd][nombre_coleccion].find().limit(limite)
    documentos = []
    for documento in cursor:
        documentos.append(json_util.dumps(documento, indent=2, ensure_ascii=False))
    return documentos


@app.route("/", methods=["GET", "POST"])
def login():
    error = None
    if request.method == "POST":
        servidor = request.form.get("servidor", SERVIDOR_POR_DEFECTO)
        puerto = int(request.form.get("puerto", PUERTO_POR_DEFECTO))
        usuario = request.form.get("usuario", "")
        contrasena = request.form.get("contrasena", "")
        auth_source = request.form.get("auth_source", "admin")
        try:
            cadena = construir_cadena(servidor, puerto, usuario, contrasena, auth_source)
            cliente = conectar(cadena)
            token = secrets.token_hex(16)
            clientes[token] = cliente
            session["token"] = token
            session["usuario"] = usuario
            session["servidor"] = servidor + ":" + str(puerto)
            session["cadena"] = "mongodb://" + quote_plus(usuario) + ":****@" + servidor + ":" + str(puerto) + "/?authSource=" + quote_plus(auth_source)
            return redirect(url_for("colecciones"))
        except PyMongoError as e:
            error = "No se pudo conectar o autenticar: " + str(e)
    return render_template(
        "login.html",
        error=error,
        servidor=SERVIDOR_POR_DEFECTO,
        puerto=PUERTO_POR_DEFECTO,
    )


@app.route("/colecciones")
def colecciones():
    cliente = obtener_cliente()
    if cliente is None:
        return redirect(url_for("login"))
    error = None
    datos = {}
    try:
        datos = colecciones_accesibles(cliente)
    except PyMongoError as e:
        error = "Error al listar las colecciones: " + str(e)
    return render_template("colecciones.html", datos=datos, error=error)


@app.route("/coleccion/<nombre_bd>/<nombre_coleccion>")
def documentos(nombre_bd, nombre_coleccion):
    cliente = obtener_cliente()
    if cliente is None:
        return redirect(url_for("login"))
    error = None
    docs = []
    try:
        docs = leer_documentos(cliente, nombre_bd, nombre_coleccion, LIMITE_DOCUMENTOS)
    except PyMongoError as e:
        error = "No tienes permiso o hubo un error: " + str(e)
    return render_template(
        "documentos.html",
        nombre_bd=nombre_bd,
        nombre_coleccion=nombre_coleccion,
        documentos=docs,
        limite=LIMITE_DOCUMENTOS,
        error=error,
    )


@app.route("/logout")
def logout():
    token = session.pop("token", None)
    if token is not None and token in clientes:
        clientes[token].close()
        del clientes[token]
    session.clear()
    return redirect(url_for("login"))


MODO_SEGURO = os.environ.get("MODO_SEGURO", "0") == "1"
PORTAL_USUARIO = os.environ.get("PORTAL_USUARIO", "portal")
PORTAL_CONTRASENA = os.environ.get("PORTAL_CONTRASENA", "portal")

portal = None


def obtener_portal():
    global portal
    if portal is None:
        cadena = "mongodb://" + quote_plus(PORTAL_USUARIO) + ":" + quote_plus(PORTAL_CONTRASENA) + "@" + SERVIDOR_POR_DEFECTO + ":" + str(PUERTO_POR_DEFECTO) + "/?authSource=admin"
        portal = MongoClient(cadena, serverSelectionTimeoutMS=3000)
    return portal


def convertir_valor(texto):
    try:
        valor = json.loads(texto)
    except ValueError:
        valor = texto
    return valor


def construir_filtro_vulnerable(nombre, pin):
    return {"nombre": convertir_valor(nombre), "pin": convertir_valor(pin)}


def construir_filtro_seguro(nombre, pin):
    filtro = None
    nombre_valido = isinstance(nombre, str) and len(nombre) > 0 and len(nombre) <= 100
    pin_valido = isinstance(pin, str) and len(pin) == 4 and pin.isascii() and pin.isdigit()
    if nombre_valido and pin_valido:
        filtro = {"nombre": nombre, "pin": int(pin)}
    return filtro


def buscar_clientes(filtro):
    cursor = obtener_portal()["tienda"]["clientes"].find(filtro, {"_id": 0})
    resultados = []
    for documento in cursor:
        resultados.append(json_util.dumps(documento, indent=2, ensure_ascii=False))
    return resultados


@app.route("/consulta", methods=["GET", "POST"])
def consulta():
    nombre = ""
    pin = ""
    filtro_mostrado = None
    resultados = []
    error = None
    buscado = False
    if request.method == "POST":
        buscado = True
        nombre = request.form.get("nombre", "")
        pin = request.form.get("pin", "")
        if MODO_SEGURO:
            filtro = construir_filtro_seguro(nombre, pin)
        else:
            filtro = construir_filtro_vulnerable(nombre, pin)
        if filtro is None:
            error = "Datos no válidos: el PIN debe tener exactamente 4 cifras"
        else:
            filtro_mostrado = json_util.dumps(filtro, ensure_ascii=False)
            try:
                resultados = buscar_clientes(filtro)
            except PyMongoError as e:
                error = "Error en la consulta: " + str(e)
    return render_template(
        "consulta.html",
        modo_seguro=MODO_SEGURO,
        nombre=nombre,
        pin=pin,
        filtro=filtro_mostrado,
        resultados=resultados,
        buscado=buscado,
        error=error,
    )


if __name__ == "__main__":
    print("Aplicación disponible en el puerto", 5000)
    app.run(host="0.0.0.0", port=5000)
```

{% raw %}
### `templates/base.html`

```html
<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Cliente MongoDB</title>
  <style>
    body { font-family: system-ui, sans-serif; background: #f4f6f5; color: #1c2b24; margin: 0; }
    header { background: #00684a; color: #fff; padding: 12px 24px; display: flex; justify-content: space-between; align-items: center; }
    header a { color: #fff; }
    main { max-width: 900px; margin: 24px auto; padding: 0 16px; }
    .tarjeta { background: #fff; border-radius: 8px; padding: 20px; box-shadow: 0 1px 3px rgba(0,0,0,.1); margin-bottom: 16px; }
    label { display: block; margin-top: 12px; font-weight: 600; }
    input { width: 100%; padding: 8px; margin-top: 4px; box-sizing: border-box; border: 1px solid #c5cfca; border-radius: 4px; }
    button { margin-top: 16px; padding: 10px 18px; background: #00684a; color: #fff; border: none; border-radius: 4px; cursor: pointer; }
    .error { background: #fde8e8; color: #9b1c1c; padding: 10px; border-radius: 4px; margin-bottom: 16px; }
    pre { background: #1c2b24; color: #e3fcf7; padding: 12px; border-radius: 6px; overflow-x: auto; }
  </style>
</head>
<body>
  <header>
    <strong>Cliente web MongoDB</strong>
    {% if session.get("usuario") %}
      <span>{{ session["usuario"] }} @ {{ session["servidor"] }} · <a href="{{ url_for('consulta') }}">Consulta de clientes</a> · <a href="{{ url_for('logout') }}">Cerrar sesión</a></span>
    {% endif %}
  </header>
  <main>
    {% if error %}<div class="error">{{ error }}</div>{% endif %}
    {% block contenido %}{% endblock %}
  </main>
</body>
</html>
```

### `templates/login.html`

```html
{% extends "base.html" %}
{% block contenido %}
<div class="tarjeta">
  <h2>Conectar con el servidor MongoDB</h2>
  <form method="post">
    <label>Servidor <input name="servidor" value="{{ servidor }}" required></label>
    <label>Puerto <input name="puerto" type="number" value="{{ puerto }}" required></label>
    <label>Usuario <input name="usuario" required></label>
    <label>Contraseña <input name="contrasena" type="password" required></label>
    <label>Base de datos de autenticación (authSource) <input name="auth_source" value="admin" required></label>
    <button type="submit">Conectar</button>
  </form>
</div>
<p><a href="{{ url_for('consulta') }}">Consulta de clientes (acceso público)</a></p>
{% endblock %}
```

### `templates/colecciones.html`

```html
{% extends "base.html" %}
{% block contenido %}
<div class="tarjeta">
  <h2>Colecciones accesibles</h2>
  <p>Cadena de conexion: <code>{{ session["cadena"] }}</code></p>
  {% if datos %}
    {% for nombre_bd, lista in datos.items() %}
      <h3>{{ nombre_bd }}</h3>
      <ul>
        {% for nombre_coleccion in lista %}
          <li><a href="{{ url_for('documentos', nombre_bd=nombre_bd, nombre_coleccion=nombre_coleccion) }}">{{ nombre_coleccion }}</a></li>
        {% endfor %}
      </ul>
    {% endfor %}
  {% else %}
    <p>Este usuario no tiene acceso a ninguna colección.</p>
  {% endif %}
</div>
{% endblock %}
```

### `templates/documentos.html`

```html
{% extends "base.html" %}
{% block contenido %}
<p><a href="{{ url_for('colecciones') }}">← Volver a las colecciones</a></p>
<div class="tarjeta">
  <h2>{{ nombre_bd }}.{{ nombre_coleccion }}</h2>
  <p>Se muestran como máximo {{ limite }} documentos. Encontrados: {{ documentos|length }}</p>
  {% for doc in documentos %}
    <pre>{{ doc }}</pre>
  {% else %}
    {% if not error %}<p>La colección está vacía.</p>{% endif %}
  {% endfor %}
</div>
{% endblock %}
```

### `templates/consulta.html`

```html
{% extends "base.html" %}
{% block contenido %}
<p><a href="{{ url_for('login') }}">← Volver al inicio</a></p>
<div class="tarjeta">
  <h2>Consulta de clientes</h2>
  <p>Introduce tu nombre y tu PIN para ver tus datos.</p>
  <form method="post">
    <label>Nombre <input name="nombre" value="{{ nombre }}" required></label>
    <label>PIN <input name="pin" value="{{ pin }}" required></label>
    <button type="submit">Consultar</button>
  </form>
</div>
{% if filtro %}
<div class="tarjeta">
  <h3>Filtro enviado a MongoDB</h3>
  <pre>{{ filtro }}</pre>
</div>
{% endif %}
{% if buscado and not error %}
<div class="tarjeta">
  <h3>Resultados: {{ resultados|length }}</h3>
  {% for doc in resultados %}
    <pre>{{ doc }}</pre>
  {% else %}
    <p>No hay ningún cliente con esos datos.</p>
  {% endfor %}
</div>
{% endif %}
{% endblock %}
```
{% endraw %}

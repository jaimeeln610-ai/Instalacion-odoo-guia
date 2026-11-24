# Guion de instalación de Odoo 18 en cualquier Windows (PowerShell)

Funciona tenga el PC lo que tenga: con o sin Python 3.12, con o sin Git, con PostgreSQL ya instalado, sin PostgreSQL pero con Docker, o sin ninguno de los dos, y con puertos ocupados. En cada paso hay una **comprobación** y, según lo que salga, la **rama** que toca.

Todo se hace en **una sola ventana de PowerShell** (si la cierras, vuelve a ejecutar el paso 0). Para instalar programas con `winget`, ábrela **como administrador**.

---

## Paso 0 · Variables (lo único que tienes que decidir)

```powershell
$ODOO   = "$env:USERPROFILE\odoo18_nuevo"   # carpeta del proyecto
$DB     = "odoo18_nuevo"                     # nombre de la base de datos
$DBUSER = "odoo"; $DBPASS = "odoo"           # usuario de PostgreSQL para Odoo
$DBPORT = 5432                               # puerto de PostgreSQL
$HTTP   = 8075; $GEV = 8076                  # puertos de Odoo (web y tiempo real)
```
- Si tu nombre de usuario de Windows tiene **espacios, tildes o ñ** (`$env:USERPROFILE` lo muestra), usa mejor `$ODOO = "C:\odoo18_nuevo"`.
- Si ya existe esa carpeta de una prueba anterior (`Test-Path $ODOO` da `True`), cambia el nombre (`odoo18_examen`) o bórrala si no te sirve.

---

## Paso 1 · Diagnóstico completo (solo mira)

```powershell
"--- Python";      py -0p 2>$null; if(-not $?){ "py no existe" }; where.exe python 2>$null
"--- PYTHONPATH";  $env:PYTHONPATH
"--- Git";         git --version 2>$null; if(-not $?){ "git no existe" }
"--- winget";      winget --version 2>$null; if(-not $?){ "winget no existe" }
"--- Docker";      docker version --format "{{.Server.Version}}" 2>$null; if(-not $?){ "Docker no instalado o no arrancado" }
"--- Servicios";   Get-Service *postgres*, *odoo* -ErrorAction SilentlyContinue | Format-Table Name,Status,StartType -AutoSize
"--- PostgreSQL";  Get-ChildItem "C:\Program Files\PostgreSQL" -ErrorAction SilentlyContinue | Select-Object Name
"--- Puertos";     Get-NetTCPConnection -State Listen -LocalPort 5432,5433,8069,8072,$HTTP,$GEV -ErrorAction SilentlyContinue |
                     Select-Object LocalPort,OwningProcess,@{n='Proceso';e={(Get-Process -Id $_.OwningProcess).ProcessName}}
"--- Odoo en marcha"; Get-CimInstance Win32_Process -Filter "Name like 'python%'" | Where-Object CommandLine -match "odoo-bin" | Select-Object ProcessId,CommandLine
"--- Contenedores"; docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}" 2>$null
"--- Disco";       Get-PSDrive C | Select-Object @{n='LibreGB';e={[math]::Round($_.Free/1GB,1)}}
```

### Qué hacer según lo que salga

| Resultado | Rama |
|---|---|
| `py -0p` muestra 3.10, 3.11 o 3.12 | Python OK → paso 2, rama A |
| Solo 3.13/3.14, o nada | Paso 2, rama B o C |
| `PYTHONPATH` tiene algo | `$env:PYTHONPATH = ""` (solo en esta ventana) |
| `git no existe` | Paso 3, rama B o C |
| `winget no existe` | Usa las ramas «sin winget» (descarga manual) |
| Servicio `postgresql-x64-NN` | PostgreSQL nativo → paso 4, rama A |
| Sin PostgreSQL y Docker responde | Paso 4, rama B |
| Sin PostgreSQL y sin Docker | Paso 4, rama C |
| Servicio `odoo-server-…` o `odoo-bin` en marcha | Paso 5 |
| Algún puerto de `$HTTP`/`$GEV` ocupado | Paso 5 |
| Menos de 3 GB libres | Libera espacio antes |

---

## Paso 2 · Python (3.10, 3.11 o 3.12)

**Rama A · Ya hay uno válido.** Elige cuál usar:
```powershell
$PY = "py -3.12"      # o "py -3.11" / "py -3.10", el que salió en py -0p
Invoke-Expression "$PY --version"
```

**Rama B · Con winget:**
```powershell
winget install -e --id Python.Python.3.12
# cierra PowerShell, ábrela otra vez, repite el paso 0
py -0p
$PY = "py -3.12"
```

**Rama C · Sin winget:** descarga el instalador de Windows de Python 3.12 desde python.org, márcale **«Add python.exe to PATH»** y deja el **py launcher** marcado. Reabre PowerShell, paso 0, y `$PY = "py -3.12"`.

> Si `py` no existe pero sí `python` 3.12 (por ejemplo, el de Microsoft Store): `$PY = "python"` y comprueba con `python --version`.

---

## Paso 3 · Git

**Rama A · Ya está** (`git --version` responde): nada que hacer.

**Rama B · Con winget:**
```powershell
winget install -e --id Git.Git
# reabre PowerShell y repite el paso 0
git --version
```

**Rama C · Sin Git y sin poder instalarlo:** en el paso 6 usarás la descarga en ZIP en vez de `git clone`.

Por si las rutas largas de Odoo dan problemas al clonar:
```powershell
git config --global core.longpaths true
```

---

## Paso 4 · PostgreSQL (elige UNA rama)

### Rama A · Ya hay PostgreSQL instalado en Windows

```powershell
Get-Service postgresql* | Format-Table Name,Status -AutoSize
Start-Service (Get-Service postgresql* | Select-Object -First 1).Name   # si estaba Stopped
$PG = (Get-ChildItem "C:\Program Files\PostgreSQL\*\bin\psql.exe" | Select-Object -Last 1).FullName
& $PG --version
```
Averigua su puerto (si hay varios PostgreSQL, cada uno tiene uno):
```powershell
Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue | Where-Object { (Get-Process -Id $_.OwningProcess).ProcessName -eq 'postgres' } | Select-Object -Unique LocalPort
$DBPORT = 5432     # pon aquí el que haya salido
```
Crea el usuario (te pide la contraseña de `postgres` que se puso al instalarlo):
```powershell
& $PG -U postgres -p $DBPORT -c "\du"
& $PG -U postgres -p $DBPORT -c "CREATE USER $DBUSER WITH CREATEDB PASSWORD '$DBPASS';"
```
- Si dice que `role "odoo" already exists`: ya está creado. Si no sabes su contraseña, ponle la tuya: `& $PG -U postgres -p $DBPORT -c "ALTER USER $DBUSER WITH CREATEDB PASSWORD '$DBPASS';"`
- Si no sabes la contraseña de `postgres`: usa la rama B (Docker) en otro puerto (`$DBPORT = 5433`).

### Rama B · Sin PostgreSQL, con Docker

Docker Desktop tiene que estar **abierto** (icono de la ballena quieto). Si el 5432 está libre:
```powershell
docker run -d --name pg-odoo18 -p "${DBPORT}:5432" -e POSTGRES_USER=$DBUSER -e POSTGRES_PASSWORD=$DBPASS -e POSTGRES_DB=postgres -v pg-odoo18-data:/var/lib/postgresql/data --restart unless-stopped postgres:16
Start-Sleep 5; docker logs pg-odoo18 --tail 3      # "database system is ready to accept connections"
```
- Si el 5432 ya está ocupado (otro PostgreSQL): `$DBPORT = 5433` y repite el `docker run`.
- Si ya existe un contenedor `pg-odoo18` de antes (`Conflict. The container name is already in use`): `docker start pg-odoo18` y úsalo. **No lo borres** si tiene datos y se creó sin `-v`.
- El usuario `odoo` ya queda creado con permiso para crear bases.

### Rama C · Sin PostgreSQL y sin Docker

```powershell
winget install -e --id PostgreSQL.PostgreSQL.16 --interactive
```
En el asistente: apunta la **contraseña de postgres**, deja el **puerto 5432** (o el que te proponga si está ocupado; anótalo en `$DBPORT`) y cierra Stack Builder sin instalar nada. Sin winget: descarga el instalador de PostgreSQL 16 para Windows (EDB) y haz lo mismo.

Después, igual que la rama A:
```powershell
$PG = (Get-ChildItem "C:\Program Files\PostgreSQL\16\bin\psql.exe").FullName
& $PG -U postgres -p $DBPORT -c "CREATE USER $DBUSER WITH CREATEDB PASSWORD '$DBPASS';"
```

### Comprobación final de PostgreSQL (cualquier rama)
```powershell
Test-NetConnection localhost -Port $DBPORT | Select-Object TcpTestSucceeded     # True
```

---

## Paso 5 · Quitar lo que choca

```powershell
# Odoo del instalador .exe
Get-Service *odoo* -ErrorAction SilentlyContinue | Stop-Service
# Odoo arrancado a mano en otra ventana
Get-CimInstance Win32_Process -Filter "Name like 'python%'" | Where-Object CommandLine -match "odoo-bin" | ForEach-Object { Stop-Process -Id $_.ProcessId }
# Contenedores de Odoo del obligatorio 1 (no borra nada)
docker stop odoo odoo-db 2>$null
# ¿Siguen ocupados mis puertos?
Get-NetTCPConnection -State Listen -LocalPort $HTTP,$GEV -ErrorAction SilentlyContinue | Select-Object LocalPort,@{n='Proceso';e={(Get-Process -Id $_.OwningProcess).ProcessName}}
```
Si `$HTTP` o `$GEV` siguen ocupados por algo que no sabes qué es, **no lo cierres**: cambia de puerto.
```powershell
$HTTP = 8077; $GEV = 8078
```

---

## Paso 6 · Descargar Odoo 18

```powershell
New-Item -ItemType Directory -Force $ODOO | Out-Null
cd $ODOO
```
**Con Git:**
```powershell
git clone https://github.com/odoo/odoo.git --depth 1 --branch 18.0 odoo
```
**Sin Git (ZIP):**
```powershell
Invoke-WebRequest https://github.com/odoo/odoo/archive/refs/heads/18.0.zip -OutFile odoo18.zip
Expand-Archive odoo18.zip -DestinationPath .
Rename-Item odoo-18.0 odoo
Remove-Item odoo18.zip
```
Comprobación:
```powershell
Test-Path "$ODOO\odoo\odoo-bin", "$ODOO\odoo\requirements.txt"    # True True
```

---

## Paso 7 · Entorno virtual y dependencias

```powershell
Invoke-Expression "$PY -m venv venv"
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
.\venv\Scripts\Activate.ps1
python --version                       # la de $PY, y el prompt empieza por (venv)
python -m pip install --upgrade pip wheel setuptools
pip install -r odoo\requirements.txt   # "Ignoring ... markers" es normal
New-Item -ItemType Directory -Force custom_addons | Out-Null
```
Tiene que terminar en **Successfully installed**. Si falla, mira la tabla de problemas.

---

## Paso 8 · odoo.conf

```powershell
@"
[options]
admin_passwd = admin
db_host = localhost
db_port = $DBPORT
db_user = $DBUSER
db_password = $DBPASS
addons_path = $ODOO\odoo\addons,$ODOO\odoo\odoo\addons,$ODOO\custom_addons
http_port = $HTTP
gevent_port = $GEV
dbfilter = ^$DB`$
"@ | Set-Content -Encoding ascii odoo.conf
Get-Content odoo.conf
```
(La comilla invertida antes del último `$` hace que se escriba `$` literal: debe quedar `dbfilter = ^odoo18_nuevo$`.)

---

## Paso 9 · Crear la base de datos

Comprueba que no existe ya:
```powershell
python -c "import psycopg2;c=psycopg2.connect(host='localhost',port=$DBPORT,user='$DBUSER',password='$DBPASS',dbname='postgres');cur=c.cursor();cur.execute('select datname from pg_database');print([r[0] for r in cur])"
```
- Si sale `odoo18_nuevo` y no la quieres, cambia `$DB`, rehaz el paso 8 y sigue. (Esta línea también confirma que usuario y contraseña funcionan.)

Inicializa (español, sin datos demo) y para al terminar:
```powershell
python odoo\odoo-bin -c odoo.conf -d $DB -i base --without-demo=all --load-language=es_ES --stop-after-init
```
Tarda uno o dos minutos. Si acaba sin `ERROR`, está.

---

## Paso 10 · Arrancar y comprobar

```powershell
python odoo\odoo-bin -c odoo.conf
```
En el log tienen que salir:
- `database: odoo@localhost:5432` (o tu puerto)
- `HTTP service (werkzeug) running on …:8075` (o tu puerto)

Abre **http://localhost:8075** · usuario `admin` · contraseña `admin`. Si salta el Firewall de Windows, permite el acceso. Los avisos de Wkhtmltopdf no importan (solo son para PDF). Deja la ventana abierta mientras uses Odoo.

---

## Paso 11 · Arrancar otro día con doble clic

Crea un script una sola vez (con la ventana de los pasos anteriores aún abierta):
```powershell
@"
cd "$ODOO"
docker start pg-odoo18 2>`$null
.\venv\Scripts\Activate.ps1
python odoo\odoo-bin -c odoo.conf
"@ | Set-Content -Encoding ascii "$ODOO\arrancar.ps1"
```
Cada día:
```powershell
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\odoo18_nuevo\arrancar.ps1"
```
(o la ruta que usaste en `$ODOO`). La línea de `docker start` no molesta si usas PostgreSQL nativo. Parar: Ctrl+C.

A mano, sin script:
```powershell
cd $env:USERPROFILE\odoo18_nuevo
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
.\venv\Scripts\Activate.ps1
python odoo\odoo-bin -c odoo.conf
```

---

## Problemas típicos

| Síntoma | Causa | Solución |
|---|---|---|
| `py : El término 'py' no se reconoce` | Python sin launcher o sin instalar | Paso 2, rama B/C; o `$PY = "python"` si es 3.10–3.12 |
| `Activate.ps1 no se puede cargar…` | Política de PowerShell | `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force` |
| `python --version` no da la del venv | venv sin activar | `.\venv\Scripts\Activate.ps1` |
| Muchos errores al compilar en `pip install` (`Microsoft Visual C++ 14.0 is required`) | Python 3.13/3.14 sin paquetes precompilados | Borra `venv` y créalo con 3.10–3.12 |
| `Filename too long` al clonar | Rutas largas | `git config --global core.longpaths true` y vuelve a clonar |
| `error during connect … dockerDesktopLinuxEngine` | Docker Desktop cerrado | Ábrelo y espera a que arranque |
| `Bind for 0.0.0.0:5432 failed: port is already allocated` | Ya hay un PostgreSQL en 5432 | Usa ese (rama A) o `$DBPORT = 5433` |
| `Conflict. The container name "/pg-odoo18" is already in use` | Contenedor de antes | `docker start pg-odoo18` |
| `Connection refused` en 5432 | PostgreSQL parado | `Start-Service postgresql-x64-16` o `docker start pg-odoo18` |
| `password authentication failed for user "odoo"` | Contraseña distinta | `ALTER USER odoo WITH PASSWORD 'odoo';` (nativo: con `$PG -U postgres`; Docker: `docker exec -it pg-odoo18 psql -U odoo -d postgres`) |
| `role "odoo" does not exist` | Usuario sin crear | `CREATE USER odoo WITH CREATEDB PASSWORD 'odoo';` |
| `permission denied to create database` | Falta CREATEDB | `ALTER USER odoo CREATEDB;` |
| `Using the database user 'postgres' is a security risk, aborting.` | `db_user = postgres` | Usa `odoo` |
| `Address already in use` / no carga en 8075 | Puerto ocupado | Paso 5; cambia `$HTTP`/`$GEV` y rehaz el paso 8 |
| `database "odoo18_nuevo" does not exist` | Falta el paso 9 | Inicializa |
| La web pide crear una base o no la encuentra | `dbfilter` no coincide | Mismo nombre en `-d` y en `dbfilter` |
| Avisos de Wkhtmltopdf | No instalado | Solo hace falta para PDF |
| Datos perdidos al recrear el contenedor | Se creó sin `-v` | Siempre con volumen; nunca `docker rm` sin copia |

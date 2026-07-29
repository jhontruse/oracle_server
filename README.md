# Oracle Database 23ai Free — Entorno local con Docker

Servidor Oracle Database 23ai Free listo para desarrollo local, levantado con Docker Compose.
Pensado para tener una base de datos Oracle en el equipo sin instalarla de forma nativa.

Imagen base: [`gvenzl/oracle-free:23-faststart`](https://github.com/gvenzl/oci-oracle-free)

---

## Requisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado y en ejecución
- ~8 GB de RAM disponibles (el contenedor está limitado a 4 GB por defecto)
- ~10 GB de espacio en disco
- Opcional: [SQL Developer](https://www.oracle.com/database/sqldeveloper/) o [DBeaver](https://dbeaver.io/) como cliente

---

## Configuración inicial

**1. Clonar el repositorio**

```bash
git clone https://github.com/jhontruse/oracle_server.git
cd oracle_server
```

**2. Crear el archivo `.env`**

Copia la plantilla y edita los valores:

```bash
cp .env.example .env
```

| Variable             | Descripción                                            | Ejemplo         |
| -------------------- | ------------------------------------------------------ | --------------- |
| `ORACLE_DATABASE`    | Nombre del PDB (pluggable database) que se crea         | `BD_JHONTRUSE`  |
| `ORACLE_PASSWORD`    | Contraseña de `SYS` y `SYSTEM`                          | —               |
| `APP_USER`           | Usuario/esquema de aplicación                           | `BD_DEVELOPER`  |
| `APP_USER_PASSWORD`  | Contraseña del usuario de aplicación                    | —               |
| `ORACLE_PORT`        | Puerto del listener (opcional, default `1521`)          | `1521`          |
| `ORACLE_EM_PORT`     | Puerto de EM Express (opcional, default `5500`)         | `5500`          |
| `ORACLE_MEM_LIMIT`   | Techo de RAM del contenedor (opcional, default `4g`)    | `4g`            |

> **Importante:** el `.env` contiene credenciales y no debe subirse al repositorio.
> Las contraseñas de Oracle deben tener al menos 8 caracteres, con mayúsculas,
> minúsculas, un número y un carácter especial.

---

## Uso

**Levantar el servidor**

```bash
docker compose up -d
```

El primer arranque tarda entre 1 y 3 minutos porque crea el PDB y el usuario.
Los arranques posteriores son mucho más rápidos.

**Seguir el proceso de arranque**

```bash
docker compose logs -f
```

Está listo cuando aparece:

```
#########################
DATABASE IS READY TO USE!
#########################
```

Sal de los logs con `Ctrl + C` — eso no detiene el contenedor.

**Verificar estado**

```bash
docker compose ps      # debe mostrar "healthy"
```

---

## Conexión

El listener está atado a `127.0.0.1`, por lo que solo acepta conexiones desde
la propia máquina. Para exponerlo a la red local, cambia `127.0.0.1` por `0.0.0.0`
en el `docker-compose.yml`.

### Usuario de aplicación (uso diario)

| Campo        | Valor                        |
| ------------ | ---------------------------- |
| Host         | `localhost`                  |
| Puerto       | `1521`                       |
| Service name | valor de `ORACLE_DATABASE`   |
| Usuario      | valor de `APP_USER`          |
| Contraseña   | valor de `APP_USER_PASSWORD` |
| Rol          | default                      |

### Administrador

| Campo        | Valor                     |
| ------------ | ------------------------- |
| Host         | `localhost`               |
| Puerto       | `1521`                    |
| Service name | `FREE`                    |
| Usuario      | `sys`                     |
| Contraseña   | valor de `ORACLE_PASSWORD` |
| Rol          | **SYSDBA**                |

> En SQL Developer usa siempre **Service name**, no SID.

**Cadena JDBC**

```
jdbc:oracle:thin:@//localhost:1521/BD_JHONTRUSE
```

**Desde terminal**

```bash
docker exec -it oracle-23ai sqlplus BD_DEVELOPER@//localhost/BD_JHONTRUSE
```

### Bases de datos disponibles

| Service name             | Descripción                                    |
| ------------------------ | ---------------------------------------------- |
| `FREE`                   | Contenedor raíz (CDB) — solo para administración |
| `FREEPDB1`               | PDB por defecto de la imagen                   |
| valor de `ORACLE_DATABASE` | PDB creado para este proyecto                |

> El usuario de `APP_USER` se crea **tanto en `FREEPDB1` como en el PDB propio**.
> Son esquemas independientes con el mismo nombre y contraseña: si creas tablas y
> parecen "desaparecer", probablemente estabas conectado al otro. Usa siempre el PDB
> definido en `ORACLE_DATABASE`.

---

## Comandos útiles

```bash
docker compose up -d              # levantar
docker compose stop               # detener (conserva el contenedor)
docker compose down               # detener y eliminar el contenedor
docker compose restart            # reiniciar
docker compose logs -f            # ver logs en vivo
docker compose ps                 # estado y healthcheck

docker exec -it oracle-23ai bash     # shell dentro del contenedor
docker exec -it oracle-23ai sqlplus / as sysdba
```

**Cambiar la contraseña de SYS y SYSTEM**

```bash
docker exec oracle-23ai resetPassword <nueva_password>
```

**Crear un usuario adicional**

```bash
docker exec oracle-23ai createAppUser <usuario> <password> <PDB_destino>
```

---

## Persistencia de datos

Los datos viven en el volumen Docker `oracle_data`, no dentro del contenedor.
Esto significa que `docker compose down` y `up` **no borran la base de datos**.

Para eliminar todo y empezar desde cero:

```bash
docker compose down -v      # ⚠️ borra el volumen y todos los datos
```

---

## Estructura del proyecto

```
oracle_server/
├── docker-compose.yml     # definición del servicio
├── .env                   # credenciales (no se versiona)
├── .env.example           # plantilla de variables
├── init-scripts/          # scripts SQL de inicialización
├── backups/               # exports y backups (se crea al arrancar)
└── README.md
```

---

## Notas de configuración

El `docker-compose.yml` incluye varios ajustes específicos para Oracle:

- **`stop_grace_period: 60s`** — Oracle necesita tiempo para un apagado limpio.
  El default de Docker (10 s) puede dejar la base inconsistente.
- **`shm_size: 2g`** — memoria compartida requerida por la instancia.
- **Puertos en `127.0.0.1`** — el listener no queda expuesto a la red local.
- **Variables con `:?`** — el compose falla con un mensaje claro si falta el `.env`,
  en vez de arrancar con una contraseña vacía.
- **Rotación de logs** — 10 MB × 3 archivos, para que no crezcan sin límite.
- **`ulimits.nofile: 65536`** — requisito de Oracle para descriptores de archivo.

---

## Problemas frecuentes

**`Cannot connect to the Docker daemon`**
Docker Desktop no está abierto.

**El contenedor se queda en `starting` o `unhealthy`**
El primer arranque tarda; el healthcheck espera hasta 180 s. Revisa `docker compose logs`.

**`ORA-12514: TNS listener does not currently know of service requested`**
El service name es incorrecto, o la base aún no terminó de arrancar.

**`ORA-01017: invalid username/password`**
Las variables del `.env` solo se aplican en la **primera** creación de la base.
Cambiarlas después no tiene efecto: usa `resetPassword` o recrea con `docker compose down -v`.

**El puerto 1521 ya está en uso**
Cambia `ORACLE_PORT` en el `.env`.

---

## Licencia

Oracle Database Free se distribuye bajo la
[Oracle Free Use Terms and Conditions](https://www.oracle.com/downloads/licenses/oracle-free-license.html).

# Infraestructura-redes-Etapa-6
Trabajo grupal en etapa 6 para entregar

## Parte 1 — Persistencia de datos (`mysql` y `dbexporter`)

### Qué hace cada servicio

**`mysql`** es la base de datos del stack. Usa la imagen `mysql:latest` y no publica
ningún puerto (`ports`), así que solo es accesible desde la red interna de Docker.

**`dbexporter`** (`prom/mysqld-exporter`) no guarda datos: se conecta a MySQL, lee
su estado interno y lo expone como métricas en el puerto 9104 para que Prometheus
las recolecte.

### Por qué la contraseña va por variable de entorno

```yaml
environment:
  MYSQL_ROOT_PASSWORD: mysecretpassword
```

La imagen oficial de MySQL necesita esta variable la primera vez que arranca para
inicializar el usuario root. Pasarla por `environment` permite configurar el
contenedor sin modificar la imagen: la misma imagen sirve para cualquier entorno
y solo cambia la configuración. En este caso el valor está escrito en texto plano
porque es un entorno de práctica; en producción iría en un archivo `.env` o en un
secret.

### Para qué se monta `my.cnf` en solo lectura

```yaml
volumes:
  - ./my.cnf:/.my.cnf:ro
command:
  - --config.my-cnf=/.my.cnf
```

`my.cnf` tiene las credenciales con las que el exporter se conecta a MySQL
(usuario, contraseña, `host=mysql`, puerto 3306). Se monta desde el host y con
`command` se le indica al exporter dónde encontrarlo. El `:ro` (read-only) está
porque el exporter solo necesita leerlo: si el contenedor fallara o fuera
comprometido, no podría modificar el archivo del host.

### Relación de dependencia

```yaml
depends_on:
  - mysql
```

El exporter no tiene sentido sin la base, por eso Compose arranca primero `mysql`.
`depends_on` solo controla el orden de inicio, no espera a que MySQL esté listo
para aceptar conexiones. Por eso el `setup.sh` tiene aparte un bucle con
`mysqladmin ping` antes de crear las tablas.

### Observación

Ninguno de los dos servicios define un volumen para los datos de MySQL
(`/var/lib/mysql`). Los datos viven dentro del contenedor: sobreviven a un
reinicio, pero se pierden si el contenedor se elimina (`docker-compose down`).

### Sintaxis YAML de esta sección

- **Indentación con espacios** (nunca tabs): define qué está dentro de qué.
  `mysql` y `dbexporter` están al mismo nivel, dentro de `services`.
- **Mapas** (`clave: valor`): `image: mysql:latest`, `restart: always`.
  `environment` acá está escrito como mapa.
- **Listas** (`- item`): `volumes`, `command`, `depends_on` y `networks`.
- **Sintaxis corta de volúmenes**: un solo string `origen:destino:modo`.

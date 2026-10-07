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

## Parte 2 — Recolección de métricas (`cadvisor` y `prometheus`)

### Qué hace cada servicio

**`cadvisor`** mide el consumo de recursos (CPU, memoria, disco, red) de todos los
contenedores que corren en el host y lo expone como métricas en su puerto 8080.

**`prometheus`** es el recolector: cada 5 segundos (`scrape_interval: 5s` en
`prometheus.yml`) consulta a cada target, y guarda las métricas como series de
tiempo. Tiene cuatro jobs: él mismo, `cadvisor`, `mysql` (vía `dbexporter`) y
`crud-app`.

### Por qué cAdvisor monta rutas del host en solo lectura

```yaml
volumes:
  - /:/rootfs:ro
  - /var/run:/var/run:ro
  - /sys:/sys:ro
  - /var/lib/docker/:/var/lib/docker:ro
  - /dev/disk/:/dev/disk:ro
```

Un contenedor está aislado y por defecto no ve a los demás. Para medirlos,
cAdvisor necesita mirar el host: `/sys` tiene los cgroups (donde el kernel lleva
la cuenta de CPU y memoria por contenedor), `/var/run` el socket de Docker,
`/var/lib/docker` el estado de los contenedores y `/dev/disk` los discos.
Todo va con `:ro` porque cAdvisor solo observa: se le da mucha visibilidad sobre
el host, así que se le quita la posibilidad de escribir.

### Qué puerto se publica y por qué

```yaml
ports:
  - "9090:9090"
```

Solo Prometheus publica un puerto (formato `host:contenedor`), porque hay que
entrar a su interfaz web desde el navegador para ver los targets y hacer consultas
PromQL. cAdvisor no publica nada: su único consumidor es Prometheus, que lo
alcanza por la red interna como `cadvisor:8080`.

### Qué resuelve `extra_hosts`

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

La app Flask del CRUD no corre en un contenedor: el `setup.sh` la levanta
directamente en el host (`nohup python3 app.py`) en el puerto 8888. Al no estar
en `mynetwork`, Prometheus no puede encontrarla por nombre de servicio.
`extra_hosts` agrega una entrada al `/etc/hosts` del contenedor para que
`host.docker.internal` apunte a la IP del host (`host-gateway`). Así funciona el
target `host.docker.internal:8888` del job `crud-app`. En Linux ese nombre no
existe por defecto, por eso hay que declararlo.

### Sintaxis YAML de esta sección

- **Strings entre comillas**: `"9090:9090"` y `"host.docker.internal:host-gateway"`
  van entre comillas para que YAML los tome como texto y no intente interpretar
  los dos puntos de otra forma.
- **Listas de strings**: `volumes`, `ports` y `extra_hosts` son listas donde cada
  item sigue el patrón `origen:destino`.
- **Versión fijada**: `cadvisor:v0.47.2` usa un tag concreto, a diferencia de
  `prometheus:latest`.
- **Rutas relativas vs. absolutas**: `./prometheus.yml` es relativa a la carpeta
  del compose; las de cAdvisor (`/sys`, `/var/run`) son absolutas del host.

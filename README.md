# Infraestructura-redes-Etapa-6
Trabajo grupal en etapa 6 para entregar

Parte 3: explico grafana y la red mynetwork (visualización y topología)

Explico por qué Grafana depende de Prometheus y cómo la red común permite que los cinco servicios se resuelvan por nombre.

Lo entendí así porque Grafana no genera datos, solo consulta a Prometheus, entonces no tiene sentido que arranque antes. La resolución por nombre la entendí siguiendo los archivos de configuración: my.cnf usa host=mysql y prometheus.yml usa cadvisor:8080 y dbexporter:9104, sin ninguna IP. El contraste me lo dio la app Flask, que al estar fuera de mynetwork necesita que setup.sh le inyecte la IP de MySQL con sed.

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

## Parte 3 — Visualización y topología (`grafana` y `mynetwork`)

### Qué hace Grafana

**`grafana`** es la capa de visualización: no recolecta ni guarda métricas, sino
que consulta a Prometheus y muestra los resultados en dashboards.

```yaml
grafana:
  image: grafana/grafana:13.2.2
  ports:
    - "3000:3000"
  environment:
    - GF_SECURITY_ADMIN_PASSWORD=admin
```

- La versión está fijada a propósito (lo aclara el comentario del archivo): la
  interfaz cambia entre releases y la guía describe pantallas concretas.
- Publica el puerto 3000 porque es el servicio que se usa desde el navegador.
- La contraseña del usuario admin se define por variable de entorno, igual que
  en MySQL.

### Por qué Grafana depende de Prometheus

```yaml
depends_on:
  - prometheus
```

Prometheus es la fuente de datos (datasource) de Grafana. Sin Prometheus, los
dashboards no tienen nada que mostrar, así que Compose lo arranca primero. Igual
que en la Parte 1, `depends_on` solo garantiza el orden de inicio.

### Cómo la red común permite resolver por nombre

```yaml
networks:
  mynetwork:
```

Este bloque, al nivel raíz del archivo, declara la red. Los cinco servicios se
conectan a ella con `networks: - mynetwork`. Al ser una red definida por el
usuario, Docker le agrega un DNS interno: cada contenedor puede encontrar a los
demás usando el nombre del servicio, sin conocer su IP.

Esto se ve en los archivos de configuración:

| Quién | A quién llama | Cómo |
|---|---|---|
| `dbexporter` | MySQL | `host=mysql` en `my.cnf` |
| `prometheus` | cAdvisor | `cadvisor:8080` en `prometheus.yml` |
| `prometheus` | dbexporter | `dbexporter:9104` en `prometheus.yml` |
| `grafana` | Prometheus | `http://prometheus:9090` al configurar el datasource |

El contraste es la app Flask: como corre fuera de la red, no puede usar el nombre
`mysql`. Por eso el `setup.sh` tiene que averiguar la IP del contenedor con
`docker inspect` y reemplazarla en `app.py` con `sed`.

De los cinco servicios, solo dos publican puertos al host (Prometheus 9090 y
Grafana 3000). El resto se comunica únicamente por `mynetwork`.

### Sintaxis YAML de esta sección

- **Claves de nivel raíz**: `version`, `services` y `networks` no tienen
  indentación; son las secciones principales del archivo.
- **Clave con valor vacío**: `mynetwork:` sin nada después significa "creá esta
  red con la configuración por defecto" (driver bridge).
- **`environment` como lista**: acá se escribe `- CLAVE=valor`, mientras que en
  `mysql` se escribió como mapa (`CLAVE: valor`). Las dos formas son válidas.
- **Comentarios**: las líneas con `#` no se ejecutan; documentan decisiones,
  como el motivo de la versión fijada.

## Conclusión

Al juntar las tres partes queda claro que el `docker-compose.yml` no es una lista
de cinco contenedores sueltos, sino una cadena: cada servicio existe porque otro
lo necesita. MySQL genera el estado, `dbexporter` y `cadvisor` lo convierten en
métricas, Prometheus las recolecta y Grafana las muestra. Si se cae un eslabón,
los que vienen después se quedan sin datos.

También hay decisiones que se repiten en las tres partes:

- **Mínimo privilegio:** todo lo que solo necesita leerse se monta con `:ro`
  (`my.cnf` en la Parte 1, las rutas del host de cAdvisor en la Parte 2).
- **Mínima exposición:** de los cinco servicios, solo se publican los dos puertos
  que se usan desde el navegador (9090 y 3000). El resto se comunica únicamente
  por `mynetwork`.
- **Configuración fuera de la imagen:** las contraseñas de MySQL y Grafana van por
  variables de entorno, y `my.cnf` y `prometheus.yml` se montan desde el host.
- **`depends_on` ordena, no espera:** aparece en `dbexporter` y en `grafana`, y en
  los dos casos solo garantiza el orden de arranque. Por eso el `setup.sh` tiene
  sus propios bucles de espera.

El punto que me terminó de unir las tres explicaciones fue la app Flask. Al correr
en el host y no en un contenedor, queda fuera de `mynetwork`, y eso explica dos
cosas que por separado parecían detalles: el `extra_hosts` de Prometheus (Parte 2)
y que el `setup.sh` tenga que inyectarle la IP de MySQL con `sed` (Parte 3). Ver lo
que pasa cuando un componente no está en la red fue lo que mejor me mostró para
qué sirve la red.

En cuanto a la sintaxis, el archivo completo se arma con tres construcciones de
YAML: mapas (`clave: valor`), listas (`- item`) e indentación con espacios para
indicar qué está dentro de qué. Una vez identificadas, cualquier sección del
archivo se puede leer de la misma manera.

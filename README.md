# turnos-booking-service

[![CI](https://github.com/ttmdz-um-progii/turnos-booking-service/actions/workflows/ci.yml/badge.svg)](https://github.com/ttmdz-um-progii/turnos-booking-service/actions/workflows/ci.yml)

Servicio Spring Boot que calcula disponibilidad y gestiona el proceso de reserva de turnos con holds, eventos Kafka e idempotencia. Tiene su propia base PostgreSQL y le pide el catálogo a `turnos-catalog-service` por su API interna, sin guardar copia.

Sigue una arquitectura hexagonal sin capa de service: los adaptadores de entrada (controladores, listener de Kafka) usan directamente los puertos de entrada, y cada tecnología externa es un adaptador en `infrastructure` del módulo que la usa.

> El esqueleto (proyecto, Compose, Flyway, health check y CI) se copió de [`turnos-catalog-service`](https://github.com/ttmdz-um-progii/turnos-catalog-service), que lo desarrolló en su épica de Setup (#1 a #5). Acá se adaptó el nombre, el package y la configuración, y se quitó Redis, que turnos no usa.

## Requisitos

- JDK 25
- Docker (con Docker Compose)

Maven no hace falta: el proyecto trae el wrapper (`./mvnw`).

## Configuración

Toda la configuración llega por variables de entorno. Para desarrollo se leen del archivo `.env` en la raíz, que no se versiona:

```bash
cp .env.example .env
```

Completar:

- `POSTGRES_USER` / `POSTGRES_PASSWORD`: superusuario del contenedor de PostgreSQL. Solo se usa para inicializar la base.
- `DB_*`: conexión de la aplicación (por ejemplo `DB_HOST=localhost`, `DB_PORT=5433`, `DB_NAME=booking`, `DB_SCHEMA=booking`). `DB_USERNAME` tiene que ser distinto de `POSTGRES_USER`: es un usuario propio, con permisos solo sobre el esquema `DB_SCHEMA`, que crea `docker/postgres/init/01-app-user.sh`.
- `APP_PORT`: puerto donde Compose publica el servicio (por ejemplo `8081`).
- `KAFKA_BOOTSTRAP_SERVERS`: Kafka de la cátedra. Puede quedar vacía por ahora.

Los puertos de ejemplo (`8081` y `5433`) difieren de los de catálogo para poder correr los dos servicios a la vez. Kafka es de la cátedra: no se levanta en este Compose.

## Levantar todo con Docker Compose

```bash
docker compose up --build
```

Levanta PostgreSQL y, cuando la base está lista, el servicio (que se compila dentro de la imagen). Comprobar que responde:

```bash
curl localhost:8081/actuator/health
```

Responde `{"status":"UP"}` mientras la base esté accesible y `{"status":"DOWN"}` (con código 503) si no lo está. Compose usa este mismo endpoint como healthcheck, así que `docker compose ps` muestra el servicio como `healthy` o `unhealthy`.

Dentro de Compose el servicio se conecta a `postgres:5432`; el resto de la configuración sale del mismo `.env`.

El usuario de la aplicación se crea solo la primera vez, con el volumen vacío. Si se cambian las credenciales o el esquema, hay que borrar la base y volver a levantarla:

```bash
docker compose down -v
```

## Desarrollo local con Maven

Levantar solo la base y correr el servicio con el wrapper, con el mismo `.env`:

```bash
docker compose up -d postgres
./mvnw spring-boot:run
```

## Tests

```bash
./mvnw verify
```

Los tests no usan la base de Compose ni el `.env`: cada corrida levanta su propio PostgreSQL con Testcontainers, así que solo hace falta Docker andando.

El CI de GitHub Actions corre lo mismo en cada PR contra `main` o `develop`, al mergear un PR en `develop` y en cada push a `main`. Un push directo a `develop` no lo dispara. Si no compila o falla un test, el check queda en rojo.

## Migraciones

El esquema de la base lo crea y lo modifica Flyway: al arrancar, el servicio aplica las migraciones pendientes de `src/main/resources/db/migration/` y anota cada una en la tabla `flyway_schema_history`. Hibernate solo valida que las entidades coincidan con las tablas.

Una migración ya aplicada no se edita, porque Flyway detecta el cambio y el servicio no arranca. Cada cambio de esquema va en un archivo nuevo con la versión siguiente, por ejemplo `V2__create_booking_table.sql`.

## Ramas

`main` está protegida: no admite push directo, solo PR. El trabajo se integra en `develop` desde ramas `feature/<n>-<tema>`.

# test-manager-library-3145556-infra

> **Prueba técnica · Ficha ADSO 3145556** — entrega: martes 13 de octubre de 2026, 17:00 (hora Colombia).

Infraestructura de *Manager Library*: contenedores, red, volúmenes y un motor PostgreSQL 16 **vacío**.
El spec vive en [`test-manager-library-3145556-docs`](../test-manager-library-3145556-docs); la traducción a Laravel/React que se usa
aquí está en `docs/spec-laravel/`.

## Reglas que sigue este repositorio

| Regla | Origen |
|---|---|
| El motor arranca vacío. Este repositorio **no tiene DDL**: ni `CREATE TABLE` ni scripts de arranque. Tablas, índices, restricciones y la semilla de categorías viven en las migraciones de la API | Constitución, artículo V; ADR-001 |
| El sistema levanta con un solo comando y la base arranca vacía | RNF-5; tarea T-20 |
| Ningún secreto en el repositorio: solo se versiona `.env.example`; `.env` se ignora | Constitución, artículo IX |
| Todas las fechas en UTC | RN-6; `data-model.md` |
| Documentación en español; código, variables y nombres de archivo en inglés | Constitución, artículo XI |

## Requisitos

- Docker con Compose v2.
- Los repositorios clonados **uno al lado del otro** (el archivo de Compose construye desde rutas relativas):

```text
alguna-carpeta/
├── test-manager-library-3145556-infra   ← este repositorio
├── test-manager-library-3145556-api     ← se construye como servicio "api"
└── test-manager-library-3145556-app     ← se construye como servicio "app" (perfil opcional)
```

## Arranque

Preparación, una sola vez: copiar el ejemplo y reemplazar cada valor `change-me`.

```bash
cp .env.example .env
```

Después, un solo comando:

```bash
docker compose up --build
```

| Servicio | Qué es | Puerto (host) |
|---|---|---|
| `db` | `postgres:16`, base `library` vacía, volumen con nombre `db-data` | `5432` |
| `api` | Se construye desde `../test-manager-library-3145556-api`. Al arrancar ejecuta las migraciones y después crea el admin inicial desde `ADMIN_USERNAME` / `ADMIN_PASSWORD` | `8000` (provisional: el spec no define el puerto) |
| `app` | Se construye desde `../test-manager-library-3145556-app`. Solo con `docker compose --profile app up --build` | `5173` |

Quién crea qué:

- `db` crea el motor, la base `library` y su usuario dueño. Nada más.
- Las migraciones de la API crean el esquema `library`, las seis tablas, las restricciones, los índices y las cinco categorías.
- El entrypoint de la API ejecuta `php artisan migrate --force` y después `php artisan library:bootstrap-admin`
  (interpretación I-08 en `docs/spec-laravel/interpretations.md`).

## Verificar que la base arranca vacía

```bash
docker compose up -d --wait db
docker compose exec db psql -U library -d library -Atc \
  "select count(*) from information_schema.tables where table_schema not in ('pg_catalog','information_schema');"
# esperado: 0
docker compose exec db psql -U library -d library -Atc "show timezone;"
# esperado: UTC
grep -rniI "create table" --exclude-dir=.git . || echo "sin DDL en infra"
```

Cuando la API haya arrancado, las consultas de `data-model.md` §10 (ejecutadas con `docker compose exec db psql -U library -d library`)
deben devolver exactamente las tablas, restricciones e índices documentados ahí.

## Estado

| Parte | Estado |
|---|---|
| `db` arranca vacío con un comando | verificado el 2026-10-13 (0 tablas, zona horaria UTC) |
| Servicio `api` | verificado el 2026-10-13: migra, crea el admin inicial y responde en `/api/v1` |
| Servicio `app` | `app` ya tiene `Dockerfile`; se levanta con `docker compose --profile app up --build` |

## Reiniciar

```bash
docker compose down -v   # elimina los contenedores y los volúmenes db-data y covers
```

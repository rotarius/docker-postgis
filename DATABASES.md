# Databases in this instance

This is a **shared** Postgres/PostGIS instance used by multiple apps in this
workspace (see [docker-compose.yml](docker-compose.yml)) - not a dedicated
database per app. This file is separate from `README.md` (the unmodified
upstream kartoza docs, kept in sync via the `upstream` remote) so it survives
future upstream merges without conflicts.

## Connection info

- From other containers on `traefik_network` (in another repo's
  `docker-compose.yml`): host `db`, port `5432`.
- From the host machine: `localhost:25432` (published in `docker-compose.yml`).
  Note: the Hetzner Cloud Firewall only allows 443 and SSH in from the
  internet, so port 25432 is not reachable from outside the server despite
  `ALLOW_IP_RANGE=0.0.0.0/0` in the container - only from the host itself or
  other containers on the same Docker host/network.
- Credentials: superuser `docker` / `docker` (see `POSTGRES_USER` /
  `POSTGRES_PASS` in `docker-compose.yml`). Same user/password work for every
  database in this instance - there is no per-app access isolation.

## Existing databases

| Database | Used by | Notes |
|----------|---------|-------|
| `gis` | - | Default database created on first init (`POSTGRES_DB`), has the PostGIS/pgRouting extensions enabled. |
| `op` | [docker-openproject](https://github.com/rotarius/docker-openproject) | `OPENPROJECT_DATABASE_URL` in its docker-compose.yml. |
| `ente_db` | [docker-ente](https://github.com/rotarius/docker-ente) | `db.name` in its `museum.yaml`. |

## Adding a database for a new app

```bash
docker compose exec db psql -U docker -d gis -c "CREATE DATABASE <name>;"
```

Then point the new app at host `db`, port `5432`, database `<name>`, using
the same `docker`/`docker` credentials - and add a row to the table above.

## Backup

The `dbbackups` service (`kartoza/pg-backup`) runs a nightly dump (default
schedule: midnight) of **every** database in the instance - its `DBLIST`
environment variable is unset, which defaults to "all databases", so new
databases are picked up automatically without any extra configuration.
Dumps are stored in the `dbbackups` volume, filenames prefixed `PG_db`
(`DUMPPREFIX`).

## Security notes

- All apps share one superuser (`docker`/`docker`), hardcoded in this
  repo's `docker-compose.yml`. Anyone with this password can read/write
  every database in the instance, not just their own app's.
- Restarting/updating this instance (e.g. for one app's needs) briefly drops
  the connection for every other app using it too.

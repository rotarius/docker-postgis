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
- The instance superuser is `docker` / `docker` (see `POSTGRES_USER` /
  `POSTGRES_PASS` in `docker-compose.yml`), used for admin tasks (creating
  databases/roles). Apps should get their own role scoped to their own
  database rather than using this superuser directly - see
  [docker-openproject](https://github.com/rotarius/docker-openproject) for
  the pattern (own `openproject` role + `openproject` database, not shared).

## Existing databases

| Database | Role | Used by | Notes |
|----------|------|---------|-------|
| `gis` | `docker` (superuser) | - | Default database created on first init (`POSTGRES_DB`), has the PostGIS/pgRouting extensions enabled. |
| `openproject` | `openproject` (scoped) | [docker-openproject](https://github.com/rotarius/docker-openproject) | `DATABASE_URL` in its `.env`. Own role, not the shared superuser. |
| `ente_db` | `docker` (superuser) | [docker-ente](https://github.com/rotarius/docker-ente) | `db.*` in its `museum.yaml`. Currently uses the shared superuser rather than a scoped role - see its README for the trade-off. |

## Adding a database for a new app

Prefer a scoped role over the shared superuser (see `openproject` above):

```bash
docker compose exec db psql -U docker -d gis -c "CREATE ROLE <app> WITH LOGIN PASSWORD '<password>';"
docker compose exec db psql -U docker -d gis -c "CREATE DATABASE <app> OWNER <app>;"
```

Then point the new app at host `db`, port `5432`, database `<app>`, user
`<app>` - and add a row to the table above.

## Backup

The `dbbackups` service (`kartoza/pg-backup`) runs a nightly dump (default
schedule: midnight) of **every** database in the instance - its `DBLIST`
environment variable is unset, which defaults to "all databases", so new
databases are picked up automatically without any extra configuration.
Dumps are stored in the `dbbackups` volume, filenames prefixed `PG_db`
(`DUMPPREFIX`).

## Security notes

- Apps using a scoped role (like `openproject`) only ever see their own
  database. Apps using the shared superuser directly (like `ente_db`
  currently does) can read/write every database in the instance, not just
  their own.
- Restarting/updating this instance (e.g. for one app's needs) briefly drops
  the connection for every other app using it too, regardless of role.

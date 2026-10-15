# Stats — media-server monitoring

Tracearr (https://github.com/connorgallopo/tracearr) monitors **Plex, Jellyfin and
Emby** from one dashboard, with watch history, transcode analytics and account-sharing
detection. Replaces `stacks/jellystat`, which covered only Jellyfin and Emby.

The stack is named for what it does, following the `yamtrack` convention.

| | |
| --- | --- |
| Public URL | https://stat.munywele.co.ke |
| Host binding | `127.0.0.1:8930` → container `3000` |
| Database | Bundled — see below |

Secrets are generated on first boot and persisted in the data volume, so there is
nothing to put in `.env` for this stack.

## Why the supervised image

Tracearr requires **TimescaleDB**. Neither Postgres in this setup provides it:

- this repository's `stacks/postgres` runs `postgres:18-alpine`
- the shared `postgres` in proxy-tool runs `postgis/postgis:17-3.5`

Installing the extension would mean adding `timescaledb` to `shared_preload_libraries`
on a database server that other applications depend on, fighting pgbouncer's
transaction pooling in the proxy-tool case, and retuning `max_locks_per_transaction`
and `nofile` for one consumer's benefit.

The `supervised` tag bundles the app, TimescaleDB and Redis into one container via
supervisord, which isolates all of that.

## Costs of this choice

| | Supervised | Standard |
| --- | --- | --- |
| Containers | 1 | 3 |
| PostgreSQL | 15 | 18 (TimescaleDB HA) |
| Minimum RAM | **3GB** | ~1GB |
| Secrets | Auto-generated | Manual `.env` |
| Support status | Bare-metal only; upstream documents it as **unsupported on VMs and nested container environments** | Documented, supported |

**Do not switch `TRACEARR_TAG` to `latest` expecting a smooth upgrade.** The
on-disk format differs between PostgreSQL 15 and 18, and PostgreSQL will refuse to
start against the old volume. Back up first:

```bash
docker exec stats node apps/server/scripts/backup.ts
docker cp stats:/data/backup ./tracearr-backups
```

Tracearr also has a built-in scheduler: Settings → Backup.

## Deploy

```bash
docker compose up -d stats
docker compose logs -f stats
```

The first start initialises PostgreSQL, so allow a couple of minutes before the UI
responds. Open https://stat.munywele.co.ke and create the first account, then
connect a media server.

## Routing

This service is proxied by **host Caddy in proxy-tool**, not by this repository's
Caddyfile, which does not carry the public media domains. The route lives at
`config/caddy/snippets/domains/personal.caddy` in proxy-tool and sets
`flush_interval -1`, because Tracearr pushes live sessions over SSE and Socket.IO
and Caddy's default response buffering would delay events.

`TRUST_PROXY=true` is set for the same reason: there is exactly one hop, this
container behind host Caddy.

## Volumes and backup

| Volume | Contents |
| --- | --- |
| `stats_postgres` | The database. The only copy of session history. |
| `stats_redis` | BullMQ job state. |
| `stats_data` | Poster image cache, application data and `/data/backup`. |

This database is **not** in this repository's Postgres stack, so it is not covered
by any dump of `postgres_data`. Back it up with Tracearr's own tooling, or add an
explicit `pg_dump` job.

## Cleaning up the old database

The superseded stack used the `jellystat` database on `stacks/postgres`. It is not
in the root compose file, so nothing creates it; drop it once you are sure you do
not need the history:

```bash
docker exec postgres dropdb --username "$POSTGRES_USER" --if-exists jellystat
```
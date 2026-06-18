# Production hardening rules

Apply **all** rules below to every service in the input compose file. Flag genuinely ambiguous cases (e.g. a service that doesn't clearly fit a tier, or unclear which service is public-facing) in the changelog's "Manual review required" section rather than guessing.

## Structural

- **[S1]** Reorder services Tier 1 → 4 (see tier definitions in SKILL.md); preserve original relative order within a tier.
- **[S2]** Alphabetize keys within every service block: `command`, `container_name`, `depends_on`, `dns`, `entrypoint`, `env_file`, `healthcheck`, `hostname`, `image`, `labels`, `logging`, `networks`, `ports`, `restart`, `stop_grace_period`, `user`, `volumes`. Keep YAML anchors/aliases syntactically valid after reordering.
- **[S3]** Remove the top-level `volumes:` block once all named volumes are converted to bind mounts (no named volumes should remain).
- **[S4]** Add the top-level `networks:` block (example below).

## Semantic renaming

- **[SR1]** Rename `redis` → `cache`; `postgres` / `mysql` / `mariadb` → `db`. Update all `depends_on`, `links`, and anchor references.
- **[SR2]** Do not rename application services or already-semantic names.

## Service identity

- **[I1]** Add `container_name: <app>-<role>` to every service.
- **[I2]** Add `hostname: <app>-<role>` to every service, matching `container_name`.

```yaml
container_name: app-postgres
hostname: app-postgres
```

## Secrets / configuration

- **[C1]** Replace every `environment:` block with `env_file`. Shared config → `config/.env`. Infra with distinct credentials → per-service files (`config/.env.db`, `config/.env.minio`, …).
- **[C2]** Remove anchors/aliases used only for env-var sharing (`&env`, `<<: *env`). Keep anchors used for `depends_on` sharing.
- **[C3]** Move image references to env vars. Convention: `DEFAULT_<ROLE>_IMAGE` for infra (e.g. `${DEFAULT_POSTGRES_IMAGE}`), `<APP>_<SERVICE>_IMAGE` for app images (e.g. `${APP_WORKER_IMAGE}`).

```yaml
# Before
environment:
  POSTGRES_USER: ${APP_DB_USERNAME}
  POSTGRES_PASSWORD: ${APP_DB_PASSWORD}

# After
env_file: config/.env.db
```

## Healthcheck

- **[H1]** Apply these shared timing values to every service's healthcheck:

```yaml
interval: 20s
retries: 3
start_interval: 5s
start_period: 60s
timeout: 15s
```

- **[H2]** `start_period: 60s` minimum for stateful services; shorter only for near-instant HTTP pings.
- **[H3]** `timeout: 15s` for stateful services; `5s` acceptable for lightweight HTTP pings.
- **[H4]** `retries: 3` for stateful services; higher only for 1s-interval lightweight checks.

Per-service `test:` commands:

| Service type | `test:` |
|---|---|
| Redis | `["CMD-SHELL", "redis-cli --raw INCR PING"]` |
| Valkey | `["CMD-SHELL", "valkey-cli --raw INCR PING"]` |
| Postgres/PGVector | `["CMD-SHELL", "pg_isready -U ${APP_DB_USERNAME}"]` |
| MariaDB/MySQL | `["CMD-SHELL", "mysqladmin ping -h localhost -u ${APP_DB_USERNAME} -p${APP_DB_PASSWORD}"]` |
| InfluxDB | `["CMD-SHELL", "curl -f http://localhost:8086/ping"]` |
| HTTP ping (ClickHouse, MinIO, …) | keep the original `test:`, apply timing values above; `timeout: 5s` acceptable |

Why the long `start_period`/`start_interval`: it prevents false-positive failures on cold starts. Fewer `retries` with a generous `timeout` avoids flapping under load.

## Logging

- **[L1]** Add a `logging` block to every service, sized by tier:

```yaml
# Tier 1 - stateless
logging:
  driver: json-file
  options:
    max-file: 2
    max-size: 2m

# Tier 2 - stateful
logging:
  driver: json-file
  options:
    max-file: 2
    max-size: 5m

# Tier 3-4 - application
logging:
  driver: json-file
  options:
    max-file: 5
    max-size: 10m
```

## Networking

- **[N1]** Assign every service to `internal-net` with a static `ipv4_address` env var.
- **[N2]** Also assign the public web service to `proxy-net`.
- **[N3]** Internal services: replace host-bound ports with container-only declarations.
- **[N4]** Public web service: remove host port bindings entirely; route via reverse-proxy labels instead.

```yaml
# Internal-only service
networks:
  internal-net:
    ipv4_address: ${APP_DB_IP}

# Public web service
networks:
  internal-net:
    ipv4_address: ${APP_INTERNAL_IP}
  proxy-net:
    ipv4_address: ${APP_IP}
```

```yaml
# Before - bound to host interface
ports:
  - 127.0.0.1:5432:5432

# After - internal-only (container port only; Docker allocates a random
# ephemeral host port, effectively unreachable without explicit forwarding)
ports:
  - 5432
```

## Reliability

- **[R1]** `restart: always` → `restart: unless-stopped` on every service.
- **[R2]** Add `stop_grace_period: 30s` to every service.

## Volumes

- **[V1]** Convert named volumes to absolute bind mounts. `/mnt/<disk>/<type>/<app>` for Tier 1–2, `/opt/persistence/<app>` for Tier 3–4. Default to `/opt/persistence/<app>` if disk layout is unknown.
- **[V2]** Preserve container-side mount paths exactly — only the host side changes.
- **[V3]** Values in the volumes section should not be enclosed in quotes.

```yaml
# Before
volumes:
  - app_valkey_data:/data
  - app_postgres_data:/var/lib/postgresql/data

# After
volumes:
  - /mnt/ssd/cache/app:/data
  - /mnt/ssd/databases/app:/var/lib/postgresql/data
```

## DNS *(public-facing web service only)*

- **[D1]** Add a `dns:` block: `127.0.0.1` first, then `${DNS_SERVER_1}`, `${DNS_SERVER_2}`.

```yaml
dns:
  - 127.0.0.1
  - ${DNS_SERVER_1}
  - ${DNS_SERVER_2}
```

## Reverse proxy *(public-facing web service only)*

- **[P1]** Add Traefik routing labels.
- **[P2]** Include commented-out `certresolver` and `domains` labels pre-filled with env vars (left commented so the user opts in deliberately once DNS/cert infra is ready).

```yaml
labels:
  - "traefik.enable=true"
  # Routers
  - "traefik.http.routers.app.entrypoints=web,websecure"
  - "traefik.http.routers.app.rule=Host(`${APP_HOST}`)"
  - "traefik.http.routers.app.tls=true"
  # - "traefik.http.routers.app.tls.certresolver=letsencrypt"
  # - "traefik.http.routers.app.tls.domains[0].main=${APP_HOST}"
  - "traefik.http.routers.app.service=app"
  # Services
  - "traefik.http.services.app.loadbalancer.server.port=3000"
ports:
  - 3000
```

## Top-level networks block

```yaml
networks:
  internal-net:
    name: internal-app
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: ${APP_NET}
          gateway: ${APP_GATEWAY}
  proxy-net:
    external: true
    name: traefik-net
```

A named bridge with explicit IPAM decouples the stack from Docker's default network naming and enables subnet-level firewall rules.

## Service ordering and naming example

```yaml
# Before
services:
  app-worker: ...
  app-web: ...
  minio: ...
  redis: ...
  postgres: ...

# After
services:
  cache: ...       # Tier 1 - was 'redis'
  db: ...          # Tier 2 - was 'postgres'
  minio: ...       # Tier 2
  app-worker: ...  # Tier 3
  app-web: ...     # Tier 4
```

## Section 4 - Application variables template

```text
APP_IMAGE=
APP_MINIO_IMAGE=
APP_WORKER_IMAGE=

APP_IP=192.168.10.x

APP_NET=192.168.x.0/24
APP_GATEWAY=192.168.x.1
APP_CACHE_IP=192.168.x.2
APP_DB_IP=192.168.x.3
APP_INTERNAL_IP=192.168.x.4
```

## Section 5 - Secrets template

```text
# ---------------------------------------------------------------------------- #
#               ------- App --------
# ---------------------------------------------------------------------------- #
APP_DB_PASSWORD=
APP_DB_USERNAME=
```

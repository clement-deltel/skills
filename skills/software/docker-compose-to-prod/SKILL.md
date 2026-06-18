---
name: docker-compose-to-prod
description: Transforms a developer-grade compose.yaml (plaintext secrets, named volumes, unbounded logs, host-exposed ports, no reverse-proxy integration) into a production-hardened version with env_file secrets, bind-mounted volumes, bounded logging, tiered service ordering, static internal-network IPs, healthchecks, graceful shutdown, and optional Traefik labels. Use whenever the user pastes a compose.yaml and asks to "harden," "productionize," "make this production-ready," "clean up," or "prep this for deployment," or requests secrets management, reverse proxy / Traefik integration, or log rotation for a compose file — even without the word "production." Also use to generate matching .env files or a changelog for a compose file.
---

# Docker Compose → Production

Deterministic transform. Apply every rule. No judgment calls beyond what's listed below.

## Input

Resolve before producing output. Do not guess.

- **App name** — default `app`.
- **Reverse proxy** — default `traefik`. If `none`: omit Traefik labels, `proxy-net`.
- **Public-facing service(s)** — receives `proxy-net` + Traefik labels + DNS. If unstated, infer from the service with a web-facing port (80/443/3000/8080) and confirm with the user before proceeding.

## Instructions

1. Inventory every service: role, `depends_on`/`links`, ports, volumes, env vars, anchors/aliases. Flag anything ambiguous under "Manual review required" — do not resolve it silently.
2. Classify into tiers. Preserve original order within each tier.
   - **Tier 1**: stateless infra — caches, brokers (Redis/Valkey, RabbitMQ)
   - **Tier 2**: stateful infra — databases, object stores (Postgres, MariaDB, InfluxDB, MinIO)
   - **Tier 3**: app workers — background processors, queue consumers
   - **Tier 4**: app web/API — public-facing
3. Apply every rule in [rules reference](references/rules.md) to every service. Read it before writing output.
4. Build top-level `networks:` — `internal-net` always; `proxy-net` (`external: true`) only if a reverse proxy is in scope.
5. Emit the five output sections below, in order. No commentary outside them.

## Output

Exactly five sections, in this order:

1. **Production compose file** — fenced YAML, complete, no placeholders, no TODOs. Top comment with upstream repo URL if identifiable.
2. **Changelog** — markdown, grouped by category (Structural, Semantic renaming, Service identity, Secrets/configuration, Healthcheck, Logging, Networking, Reliability, Volumes, DNS, Reverse proxy), one bullet per rule applied. End with "Manual review required" listing all flagged ambiguities.
3. **Environment files** — each `env_file` as a separate fenced block labeled with its path. Include every variable inferable from the original `environment:` blocks.
4. **Application variables** — non-sensitive (images, network/gateway/IPs), `KEY=` format, per template in `references/rules.md`.
5. **Secrets** — sensitive only (passwords, credentialed usernames, keys, tokens), under a banner naming the app, `KEY=` format.

## Fixed decisions

- **Volume paths**: `/mnt/<disk>/<type>/<app>` for Tier 1–2. `/opt/persistence/<app>` for Tier 3–4. Unknown disk layout → default to `/opt/persistence/<app>`.
- **env_file split**: one shared `config/.env` per app (web+worker). Separate file per infra service with distinct credentials (`config/.env.db`, `config/.env.minio`).
- **Anchors/aliases**: drop env-var-sharing anchors. Keep `depends_on`-sharing anchors.
- **Public service ambiguous**: ask. Do not pick the most plausible candidate.
- **Missing healthcheck on Tier 3–4 service**: apply standard timing regardless. Infer a real probe (e.g. `wget -qO- http://localhost:<port>/ || exit 1` for an HTTP service, a process check for a worker). Flag as assumed under "Manual review required." Never substitute a no-op command (`echo ok`, `exit 0`) — it always passes and hides failures.

Read `references/rules.md` now for the full rule table and worked examples (healthcheck timing, logging options, port binding, volume conversion, Traefik labels, top-level networks). Apply the exact values given — do not approximate from memory.

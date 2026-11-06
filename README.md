# bookorbit

BookOrbit — self-hosted reading space (ebooks, PDFs, audiobooks, comics,
podcasts) with a web reader, multi-user support, OPDS and Kobo sync.

Built from the upstream `docker-compose.yml` (v3.6.0) as a stackbuilder
project: `app/` holds the components, `app/build/` the generated
per-environment compose files.

## Quick start

```bash
cp app/build/forwarding/.env.example .env   # fill JWT_SECRET / SETUP_BOOTSTRAP_TOKEN
docker compose up -d
```

`docker compose` picks up `docker-compose.db.yml` via `COMPOSE_FILE`
(see `.env.example`). The database is PostgreSQL 18 with pgvector
(`pgvector/pgvector:pg18`); the app-mandated extensions (`uuid-ossp`,
`pg_trgm`, `unaccent`, `vector`) are created on first init.

Environments: `forwarding` (host port), `letsencrypt`, `step-ca`
(reverse-proxy networks, no host port). Rebuild with
`sb build` from `app/`.

See `DEPLOYMENT.md` for the deployment model and staged extension plan
(oidc next; pgrust swap at Stage A2), `UPSTREAM.md` for provenance.

# Deployment model — bookorbit

Upstream: https://github.com/bookorbit/bookorbit @ v3.6.0
(commit a96ff0c85a4c25e9126dfa4c71ea87537a3baf7a)
Image: `ghcr.io/bookorbit/bookorbit` (published; read-aloud worker also on ghcr)
Analysis clone: /tmp/opencode/upstreams/bookorbit (kept for stack-init Phase 1)

## 1. Product parts (inventory)

| part | type | evidence |
|------|------|----------|
| bookorbit (single container) | product service | NestJS server + Vue client in ONE image, `docker-compose.yml` service `app`, image `${APP_IMAGE?required}` (published as `ghcr.io/bookorbit/bookorbit`), port 3000, `read_only: true` + tmpfs, env-driven config; web UI, readers (EPUB/PDF/comics), audiobooks, podcasts, stats, multi-user, OPDS, Kobo sync all in-app |
| postgres database | required infra, separate container | upstream compose service `postgres`, image `pgvector/pgvector:pg18`; app REQUIRES pgvector (`vector`) plus `uuid-ossp`, `pg_trgm`, `unaccent` (docker/postgres/init.sql; docs: "PostgreSQL 18 + pgvector") |
| kokoro TTS | opt-in product companion | upstream compose profile `tts`, `ghcr.io/remsky/kokoro-fastapi-cpu:v0.8.0`, internal-only 8880; app calls `http://kokoro:8880/v1` (OpenAI-compatible TTS) |
| read-aloud-worker | opt-in product companion | upstream compose profile `read-aloud`, `ghcr.io/bookorbit/bookorbit-read-aloud-worker:latest`, internal-only 8787, `depends_on: app healthy`, mounts `/books:ro` + shared `/data/read-aloud`, Bearer-token auth |
| KOReader plugin | client-side artifact | `koreader-plugin/` in repo — installs on the KOReader device, not a deployable compose part |

Volumes: `/books` (book library, local filesystem) and `/data` (app data) on the
app; `/var/lib/postgresql/data` on the DB.

## 2. Interaction map

- browser → bookorbit:3000 (HTTP): web UI; auth = local accounts + JWT
  (first-run setup protected by `SETUP_BOOTSTRAP_TOKEN`) or OIDC/SSO
  (native support for Authentik/Keycloak/Authelia-style providers;
  `DISABLE_LOCAL_AUTH` can force SSO-only).
- bookorbit → bookorbit-db:5432 (in-group): Postgres over `DATABASE_URL`
  (or `POSTGRES_*`), Drizzle migrations, needs the four extensions above.
- bookorbit → kokoro:8880/v1 (in-network, internal): TTS requests when the
  `tts` profile is on.
- bookorbit → read-aloud-worker:8787 (in-network, internal, Bearer token):
  read-aloud job dispatch when `read-aloud` profile is on; worker reads
  `/books` read-only and writes its own `/data/read-aloud`.
- reader apps / devices → bookorbit: OPDS feeds, Kobo sync, send-to-Kindle
  (in-part endpoints).
- bookorbit outbound: 14 metadata providers, Hardcover/Readwise/StoryGraph
  sync, podcast feed fetching (all in-part, env/API-key configured).

Breakage: without postgres the app cannot start (migrations + extensions);
absent kokoro/read-aloud-worker only the TTS features degrade — app runs.

## 3. Functional base

Product parts: **bookorbit** single core product service.

Groups: one group — bookorbit + its database (simple component, never split
out; DB is in-group, no cross-host link).

Mandatory infra verdicts (always attempted):

| infra | verdict | reason |
|-------|---------|--------|
| database | **REQUIRED — as `database/postgresql` stage** | Drizzle migrations + app-mandated extensions (`uuid-ossp`, `pg_trgm`, `unaccent`, `vector`). Documented incompatibility with the collection default `postgres:alpine` (no pgvector): the postgresql stage pins `pgvector/pgvector:pg18` (upstream-pinned major 18 justified by the pgvector requirement). Wired per the §8 dual-file extension pair. |
| database/pgrust | **WORKS — tested at Stage A2** | Hypothesis of a `vector` refusal was wrong: `malisper/pgrust:v0.2` reports "pgrust 0.2 (PostgreSQL 18.3 compatible)" and ships pgvector — `init-extensions.sql` enabled all four app-mandated extensions (`uuid-ossp`, `pg_trgm`, `unaccent`, `vector`; verified via `SELECT extname FROM pg_extension`). App booted healthy on it (Nest started, HTTP 200). postgresql stage remains the default (upstream-pinned pgvector image); pgrust is a validated drop-in swap. |
| OIDC | **WORKS — tested at Stage B (browser SSO gate passed)** | Native OIDC/SSO support upstream. Provider config is stored in the DB via the admin settings UI (Settings → SERVER → Users & Access → OIDC / SSO → Add Provider) — **no env vars for issuer/clientId/secret**. Kanidm client `bookorbit`, redirect `https://bookorbit.af/oauth2-callback`, scopes `openid profile email groups`. Two requirements discovered: `OIDC_ALLOW_LOCAL_ISSUERS=true` is REQUIRED (app throws `PrivateAddressException` on the private-IP kanidm host otherwise), and the provider's **"Auto-provision users" switch must be enabled** in the provider edit form (first SSO attempt without it hit "Your account has not been set up"). Full flow verified: login button → Kanidm auth → consent → callback → auto-provisioned user → active dashboard. `DISABLE_LOCAL_AUTH` available for SSO-only once validated. |
| s3-storage | **REFUSED** | No object-storage abstraction anywhere upstream (no s3/boto/minio hits); book files and app data are local-filesystem only (`/books`, `/data` volumes). |

Functional base = 1× `bookorbit` service (`ghcr.io/bookorbit/bookorbit`) +
volumes (`bookorbit-books:/books`, `bookorbit-data:/data`, DB volume) +
required env `JWT_SECRET`, `SETUP_BOOTSTRAP_TOKEN`, `APP_URL`/`CLIENT_URL`,
DB credentials (all overridable per §5) + the database/postgresql extension.

Cross-part functional gate scenario (Stage A):

1. `docker compose up -d` → app healthy, db healthy.
2. Open UI → first-run setup wizard completed with `SETUP_BOOTSTRAP_TOKEN` →
   admin account created, migrations applied (extensions present).
3. Add a book (upload / scan library) → parsed metadata + cover.
4. Open the book in the web reader, progress recorded (stats).
5. Exercise search incl. fuzzy/semantic paths that hit `pg_trgm`/`vector`.
6. `docker compose restart` → library + progress persist on volumes.

## 4. Base stage plan

1. **Stage A (core group + DB)**: `app/` compose — service `bookorbit`
   (ports `${...}:3000`, volumes above), plus the `database/postgresql`
   extension pair wiring (`docker-compose.db.yml` with the
   `pgvector/pgvector:pg18` image, `pg_isready` healthcheck, app
   `depends_on: service_healthy`). Run the gate scenario above.
2. **Stage A2 (pgrust swap)**: swap `database/pgrust` into the build; if it
   fails on missing `vector` extension → revert, record refusal. **Result:
   WORKS** — pgrust v0.2 ships pgvector; all four app-mandated extensions
   enabled, app healthy (see §3 verdict table).
3. **Stage B (oidc)**: OIDC wiring + browser SSO gate; keep the tested
   no-oidc variant as proof of the pre-OIDC step. **Result: WORKS** —
   provider configured via the admin settings UI against kanidm
   (`OIDC_ALLOW_LOCAL_ISSUERS=true` + auto-provision toggle required, see
   §3); full browser SSO flow passed (login → Kanidm auth → consent →
   callback → auto-provisioned user dashboard).
4. No further groups → no `step-ca-trust` stages, no cross-group link gates.

## 5. Extension catalog (nothing implemented without confirmation except mandatory infra)

| extension | type | status | dependencies |
|-----------|------|--------|--------------|
| oidc | infra (mandatory per §10) | **works (tested at Stage B)** | provider (kanidm); `OIDC_ALLOW_LOCAL_ISSUERS=true` + auto-provision toggle |
| database/pgrust | infra | **works (tested at A2)** | swaps postgresql at A2 |
| tts-kokoro | product companion | **works (tested at extension stage)** | none; internal 8880 |
| read-aloud-worker | product companion | available | needs same books volume mounted ro + own read-aloud data dir |
| koreader-plugin | client-side | available (document only, not compose) | KOReader device; needs public app URL |
| s3-storage | infra | **refused** | no upstream S3 support |

## 6. Open hypotheses (validate at base stage)

- OIDC wiring surface: **RESOLVED at Stage B** — provider config lives in the
  DB (`app_settings` key `oidc_config`), entered via the admin settings UI;
  no env vars for issuer/clientId/secret. Only `OIDC_ALLOW_LOCAL_ISSUERS`
  (extension fragment, default false) is env-wired.
- Healthcheck endpoint for the app image: identify an in-container check
  (curl/wget presence unknown) — compose may need a node-based check
  (confidence: medium; observe at Stage A).
- App user permission model: `read_only` container with PUID/PGID and
  capability adds — verify the named-volume flows (books writable by app,
  read-aloud worker isolation) (confidence: medium).
- Semantic search: whether pgvector embeddings are generated in-app without
  an external LLM key (confidence: low; observe whether feature needs extra
  env in the gate).
- Kokoro/read-aloud stay out of the base: upstream treats both as opt-in
  profiles; gate does not exercise them (confidence: high).

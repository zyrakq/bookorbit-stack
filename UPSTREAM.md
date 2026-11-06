# Upstream provenance

## app — https://github.com/bookorbit/bookorbit

| date | upstream commit | image tag / digest | note |
|------|-----------------|--------------------|------|
| 2026-11-06 | a96ff0c85a4c25e9126dfa4c71ea87537a3baf7a | ghcr.io/bookorbit/bookorbit:3.6.0 | initial dockerization; GHCR tags have no `v` prefix (release.yml uses `version`, not `tag`) |
| 2026-11-06 | a96ff0c85a4c25e9126dfa4c71ea87537a3baf7a | ghcr.io/remsky/kokoro-fastapi-cpu:v0.8.0 | tts-kokoro extension fragment added (upstream `tts` profile service, internal-only 8880) |
| 2026-11-06 | a96ff0c85a4c25e9126dfa4c71ea87537a3baf7a | ghcr.io/bookorbit/bookorbit-read-aloud-worker:latest | read-aloud-worker extension fragment added (upstream `read-aloud` profile service + `docs/READ_ALOUD.md`: worker env `READ_ALOUD_WORKER_TOKEN(_FILE)`/`READ_ALOUD_LIBRARY_ROOTS`/`READ_ALOUD_WORK_ROOT`, app env `READ_ALOUD_WORKER_URL`/`READ_ALOUD_MAX_OUTPUT_BYTES`, internal-only 8787, `depends_on: app healthy`) |
| 2026-11-06 | a96ff0c85a4c25e9126dfa4c71ea87537a3baf7a | ghcr.io/bookorbit/bookorbit:3.6.0 | koreader-plugin extension consulted: client-side Lua artifact (`koreader-plugin/bookorbit.koplugin/`) installed into device `koreader/plugins/`; server counterpart (`server/src/modules/koreader/`, `/koreader/*` sync API + preconfigured plugin zip) already inside the app image — no deployable compose part, doc-only status |

# vertice-local

Docker Compose orchestration for the vertice stack. **No application code lives in this repo** —
it only wires together three sibling repos, which must be checked out alongside this one:

**This repo is a candidate for deprecation.** Each sibling app now runs standalone via its own
`npm run dev` / `./gradlew bootRun`, with env defaults already wired to talk to each other over
`localhost` (see each repo's own README/CLAUDE.md for its native run instructions, and this
repo's README for the combined native walkthrough). Docker Compose here remains as an
alternative single-command path, but native runs avoid the image-cache/rebuild pain that
motivated this split — prefer them for day-to-day work.

```
Workspace/
├── vertice-local/       (this repo — compose config only)
├── vertice-web-react/   (Next.js frontend)
├── vertice-bff/         (Node/tsx backend-for-frontend)
└── vertice-api/         (Spring Boot API)
```

Application changes (features, bugs, tests) belong in those sibling repos, not here. This repo is
the right place for: compose service definitions, env var wiring between services, port mappings,
and volume/cache configuration.

## Commands

```sh
cp .env.example .env       # first time only
docker compose up --build  # start everything
docker compose down        # stop (Ctrl-C then `down` also works)
docker compose down -v     # also wipes Postgres volume + Gradle/npm caches
```

No lint/test/build commands exist in this repo — those live in the sibling repos.

## Architecture

`vertice-web-react` has no `Dockerfile` and is **not** part of this compose file — it runs
natively only (`npm run dev`, `http://localhost:5173`), since it was the piece most affected by
Docker image-cache/rebuild pain. This repo's `docker-compose.yml` only covers Postgres,
`vertice-api`, and `vertice-bff`.

Startup order (`depends_on` + healthchecks, not just declaration order):
`postgres` → `vertice-api` (Flyway migrations run automatically) → `vertice-bff`. Start
`vertice-web-react` yourself, separately, once `bff` is up.

Source in both app repos is bind-mounted for hot reload:
- `vertice-bff` — `tsx watch` restarts on save
- `vertice-api` — Gradle `--continuous` + `spring-boot-devtools` recompiles/restarts on save

| Service | URL | Notes |
|---|---|---|
| web (native, not compose) | http://localhost:5173 | run separately: `cd ../vertice-web-react && npm run dev` |
| bff | http://localhost:3000 | proxies to api over gRPC |
| api (HTTP) | http://localhost:8080 | health: `/actuator/health` |
| api (gRPC) | localhost:9090 | used by bff, not exposed externally |
| postgres | localhost:5432 | |

`vertice-api` also ships its own standalone `docker-compose.yml` (Postgres only), for running the
API natively against a containerized DB without the rest of this stack.

## Environment (`.env`, see `.env.example`)

| Var | Default | Used by |
|---|---|---|
| `DB_NAME` | `vertice` | postgres, api |
| `DB_USERNAME` | `vertice` | postgres, api |
| `DB_PASSWORD` | `vertice` | postgres, api |
| `BFF_LOG_LEVEL` | `info` | bff |
| `JWT_SECRET` | `dev-secret-change-me` | bff |
| `JWT_EXPIRES_IN` | `8h` | bff |

`bff`'s `CORS_ORIGIN` is hardcoded in `docker-compose.yml` (not env-driven) since it's fixed to
`vertice-web-react`'s port from the table above.

## Gotchas

- `node_modules` (bff) and the Gradle cache (api) are **named volumes**, not bind mounts —
  intentional, to avoid slow bind-mounted `node_modules` on macOS and keep caches warm across
  restarts. `docker compose down` (without `-v`) preserves them.
- On macOS, Docker Desktop must use **VirtioFS** (Settings → General → file sharing
  implementation) — default on modern Docker Desktop, much faster than legacy gRPC-FUSE for the
  bind-mounted source dirs here.
- Give Docker Desktop ≥4 CPUs / ≥8GB RAM — a JVM plus a Node dev server is memory-hungry, and
  Gradle's first build (dependency download) is CPU-heavy.
- Only share `~/Workspace` (or a narrower path containing these repos) with Docker Desktop's file
  sharing — avoid sharing the whole home directory.

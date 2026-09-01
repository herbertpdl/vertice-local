# vertice-local

Docker Compose orchestration for the vertice stack. **No application code lives in this repo** —
it only wires together three sibling repos, which must be checked out alongside this one:

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

Startup order (`depends_on` + healthchecks, not just declaration order):
`postgres` → `vertice-api` (Flyway migrations run automatically) → `vertice-bff` → `vertice-web-react`.

Source in all three app repos is bind-mounted for hot reload:
- `vertice-web-react` — Next.js Fast Refresh (instant)
- `vertice-bff` — `tsx watch` restarts on save
- `vertice-api` — Gradle `--continuous` + `spring-boot-devtools` recompiles/restarts on save

| Service | URL | Notes |
|---|---|---|
| web | http://localhost:5173 | |
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

`bff`'s `CORS_ORIGIN` and `web`'s `NEXT_PUBLIC_API_BASE_URL` are hardcoded in `docker-compose.yml`
(not env-driven) since they're fixed to the local port layout above.

## Gotchas

- `node_modules` (bff/web) and the Gradle cache (api) are **named volumes**, not bind mounts —
  intentional, to avoid slow bind-mounted `node_modules` on macOS and keep caches warm across
  restarts. `docker compose down` (without `-v`) preserves them.
- On macOS, Docker Desktop must use **VirtioFS** (Settings → General → file sharing
  implementation) — default on modern Docker Desktop, much faster than legacy gRPC-FUSE for the
  bind-mounted source dirs here.
- Give Docker Desktop ≥4 CPUs / ≥8GB RAM — a JVM plus two Node dev servers is memory-hungry, and
  Gradle's first build (dependency download) is CPU-heavy.
- Only share `~/Workspace` (or a narrower path containing these repos) with Docker Desktop's file
  sharing — avoid sharing the whole home directory.

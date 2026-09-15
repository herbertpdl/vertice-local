# vertice-local

Local dev orchestration for the vertice app: `vertice-web-react`, `vertice-bff`, and `vertice-api`
must be checked out as sibling directories to this one:

```
Workspace/
├── vertice-local/       (this repo)
├── vertice-web-react/
├── vertice-bff/
└── vertice-api/
```

## Recommended: run each app natively

Docker image caching/rebuilds (especially for `vertice-web-react`) make the Compose flow below
slow for day-to-day development. Each app now runs standalone, connecting to the others over
plain `localhost` ports, with no Docker required except for Postgres. One terminal per app:

```sh
# 1. vertice-api — Postgres (Docker) + the API natively
cd ../vertice-api
docker compose up -d
./gradlew bootRun --args='--spring.profiles.active=local'
# optional, in a second terminal, for hot reload on save:
./gradlew build --continuous -x test

# 2. vertice-bff
cd ../vertice-bff
cp .env.example .env   # first time only
npm install
npm run dev

# 3. vertice-web-react
cd ../vertice-web-react
cp .env.local.example .env.local   # first time only
npm install
npm run dev
```

| Service | URL |
|---|---|
| web | http://localhost:5173 |
| bff | http://localhost:3000 |
| api (HTTP) | http://localhost:8080 (health: `/actuator/health`) |
| api (gRPC) | localhost:9090 |
| postgres | localhost:5432 |

Each app's own env defaults already point at these ports, so no extra wiring is needed —
`vertice-web-react`'s dev server is pinned to `5173` (not Next's usual `3000`) since `3000` is
`vertice-bff`'s default, and `vertice-bff`'s `CORS_ORIGIN`/`vertice-api`'s gRPC target already
default to match. See each repo's own README for details and troubleshooting.

Given this, **this repo is a candidate for deprecation** — everything it does can now be done by
running each app on its own, without needing a fourth checkout to hold the orchestration.

## Alternative: Postgres + API + BFF via Docker Compose

`vertice-web-react` no longer has a `Dockerfile` — it runs natively only, since it was the piece
most affected by Docker image-cache/rebuild pain. This compose file now only covers Postgres,
`vertice-api`, and `vertice-bff`; still start `vertice-web-react` yourself (`npm install && npm run dev`,
`http://localhost:5173`) alongside it.

```sh
cp .env.example .env   # first time only
docker compose up --build
```

This starts, in order: Postgres → `vertice-api` (Spring Boot, Flyway migrations run automatically)
→ `vertice-bff`. Source in both app repos is bind-mounted, so edits hot-reload:

- `vertice-bff` — `tsx watch` restarts on save
- `vertice-api` — Gradle `--continuous` + `spring-boot-devtools` recompiles and restarts on save

Stop with `docker compose down` (Ctrl-C then `down` also works). Add `-v` only if you want to wipe
the Postgres volume and Gradle cache.

### Notes

- `node_modules` for `bff` and the Gradle cache for `api` are named Docker volumes, not bind
  mounts — this avoids slow bind-mounted `node_modules` on macOS and keeps dependency caches warm
  across restarts (`docker compose down` without `-v` preserves them).
- On macOS, make sure Docker Desktop is using **VirtioFS** (Settings → General → file sharing
  implementation) — it's the default on modern Docker Desktop and is significantly faster than the
  legacy gRPC-FUSE backend for the bind-mounted source directories used here.
- Give Docker Desktop enough resources (Settings → Resources): ≥4 CPUs / ≥8GB RAM recommended,
  since a JVM plus a Node dev server running concurrently is memory-hungry, and Gradle's first
  build (downloading dependencies) is CPU-heavy.
- Only `~/Workspace` (or a narrower path containing these repos) needs to be shared with Docker
  Desktop's file sharing — avoid sharing your whole home directory.
- `vertice-api` also has its own standalone `docker-compose.yml` (Postgres only) if you want to run
  the API natively against a containerized DB without the rest of this stack — this is also what
  the native workflow above uses.

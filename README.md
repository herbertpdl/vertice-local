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

## Usage

```sh
cp .env.example .env   # first time only
docker compose up --build
```

This starts, in order: Postgres → `vertice-api` (Spring Boot, Flyway migrations run automatically)
→ `vertice-bff` → `vertice-web-react`. Source in all three app repos is bind-mounted, so edits hot-reload:

- `vertice-web-react` — Next.js Fast Refresh (instant)
- `vertice-bff` — `tsx watch` restarts on save
- `vertice-api` — Gradle `--continuous` + `spring-boot-devtools` recompiles and restarts on save

| Service | URL |
|---|---|
| web | http://localhost:5173 |
| bff | http://localhost:3000 |
| api (HTTP) | http://localhost:8080 (health: `/actuator/health`) |
| api (gRPC) | localhost:9090 |
| postgres | localhost:5432 |

Stop with `docker compose down` (Ctrl-C then `down` also works). Add `-v` only if you want to wipe
the Postgres volume and Gradle/npm caches.

## Notes

- `node_modules` for `bff`/`web` and the Gradle cache for `api` are named Docker volumes, not bind
  mounts — this avoids slow bind-mounted `node_modules` on macOS and keeps dependency caches warm
  across restarts (`docker compose down` without `-v` preserves them).
- On macOS, make sure Docker Desktop is using **VirtioFS** (Settings → General → file sharing
  implementation) — it's the default on modern Docker Desktop and is significantly faster than the
  legacy gRPC-FUSE backend for the bind-mounted source directories used here.
- Give Docker Desktop enough resources (Settings → Resources): ≥4 CPUs / ≥8GB RAM recommended,
  since a JVM plus two Node dev servers running concurrently is memory-hungry, and Gradle's first
  build (downloading dependencies) is CPU-heavy.
- Only `~/Workspace` (or a narrower path containing these repos) needs to be shared with Docker
  Desktop's file sharing — avoid sharing your whole home directory.
- `vertice-api` also has its own standalone `docker-compose.yml` (Postgres only) if you want to run
  the API natively against a containerized DB without the rest of this stack.

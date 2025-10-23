# Copilot instructions for this repo

Use this as your quick-start mental model and playbook to be productive in this codebase.

## Big picture
- Monorepo managed by pnpm + Turborepo. Frontends are Next.js apps in `apps/{web,admin,space}`; a realtime collab server in `apps/live` (Node/TypeScript); the backend API is Django in `apps/api` (separate Python world, not part of pnpm workspace).
- Service boundaries and URLs (see `apps/api/plane/urls.py`):
  - Web app API (session auth): `GET/POST /api/...` (namespace: plane.app)
  - Public/space endpoints: `/api/public/...`
  - External API (API key auth): `/api/v1/...` (namespace: plane.api). Auth via `X-Api-Key` header.
  - Auth endpoints: `/auth/...`
- Shared TS packages live under `packages/*` (design system, types, services, utils, i18n, state). Frontends consume these via `@plane/*` workspaces.

## Local development workflows
- Requirements: Node >= 22.18, pnpm 10.x (see root `package.json` engines and `packageManager`).
- Frontends: from repo root, run `pnpm dev` (Turborepo). Default ports: web 3000, admin 3001, space 3002 (see each app's `package.json`). Env is read from `NEXT_PUBLIC_*` vars (see `packages/constants/src/endpoints.ts`).
- Live server: `apps/live` uses `tsdown`; dev script runs build-on-change then `node dist/start.js` with `.env`.
- Backend: use `docker-compose-local.yml` for Postgres, Valkey/Redis, RabbitMQ, MinIO, Django API, Celery `worker` and `beat-worker`, plus a one-off `migrator`. API container uses `apps/api/.env` and starts via `./bin/docker-entrypoint-*.sh`.
- Type checks/lint/format (JS/TS): `pnpm check` / `pnpm check:types` / `pnpm check:lint` / `pnpm check:format` (wired via Turbo). Python linting is configured in `apps/api/pyproject.toml` (ruff).
- Tests (Python API): from `apps/api`, `python -m pytest` or use `./run_tests.py` to select markers (`-u` unit, `-c` contract, `-s` smoke), `-p` parallel, `-o` coverage (script enforces 90% via coverage report). Test structure and fixtures: `apps/api/plane/tests/README.md`.

## Conventions and patterns
- Client-service pattern for HTTP in TS:
  - Base class `packages/services/src/api.service.ts` wraps axios with `withCredentials: true`. Subclass per domain (see folders in `packages/services/src/*`). Use `API_URL` from `@plane/constants` built from `NEXT_PUBLIC_API_BASE_URL` + `NEXT_PUBLIC_API_BASE_PATH`.
  - Realtime: `packages/services/src/live.service.ts` extends `APIService`; `apps/live` uses Hocuspocus (Yjs) with Redis.
- Environment wiring used across apps (see `turbo.json` `globalEnv` and `packages/constants/src/endpoints.ts`): `NEXT_PUBLIC_{API,ADMIN,SPACE,WEB,LIVE}_{BASE_URL,BASE_PATH}`, analytics (`NEXT_PUBLIC_POSTHOG_*`), and flags.
- API documentation helpers: prefer modular OpenAPI utilities in `apps/api/plane/utils/openapi/*` and import via `from plane.utils.openapi import ...`. Decorators like `workspace_docs`, `project_docs`, `issue_docs`, `asset_docs` standardize schema. DRF Spectacular can be enabled with `ENABLE_DRF_SPECTACULAR`.
- URL namespaces: when reverse-resolving in Django tests/views, use `reverse("api:...")` for external API (`/api/v1/`) and `reverse("...")` for web app API (`/api/`) as noted in tests README.
- Workspaces/pinning: pnpm workspace is defined in `pnpm-workspace.yaml` (note `!apps/api` is excluded). Versions are catalog-pinned (e.g., `next`, `typescript`, `vite`, etc.).

## Typical change flows
- Add a new REST endpoint (backend → frontend):
  1) Backend: add DRF view/serializer under `apps/api/plane/api/...`, register URL under `/api/` (web) or `/api/v1/` (external) in `apps/api/plane/urls.py` and use OpenAPI decorators from `plane.utils.openapi`.
  2) Tests: add pytest contract tests under `apps/api/plane/tests/contract/{app|api}/` using the right client fixture (`session_client` for `/api/`, `api_key_client` for `/api/v1/`).
  3) Frontend service: create/update a small wrapper in `packages/services/src/<domain>/...` extending `APIService` and using `API_URL`.
  4) UI: consume the service from `apps/{web,admin,space}`; prefer shared types in `@plane/types` and UI primitives in `@plane/ui`.

## Handy file map
- Repo orchestration: `package.json`, `turbo.json`, `pnpm-workspace.yaml`
- Frontends: `apps/web`, `apps/admin`, `apps/space` (Next.js), `apps/live` (Node/TS server)
- Backend: `apps/api/plane/urls.py`, `apps/api/plane/*` (Django), compose: `docker-compose-local.yml`
- Services and constants: `packages/services/src/*`, `packages/constants/src/endpoints.ts`
- Tests and helpers: `apps/api/pytest.ini`, `apps/api/run_tests.py`, `apps/api/plane/tests/README.md`, OpenAPI: `apps/api/plane/utils/openapi/*`

If anything here is unclear or you’re missing a decision/rule of thumb you expect in this repo, call it out so we can refine these instructions.
# AGENTS.md

Compact guide for OpenCode sessions. See `INSTRUCTIONS.md` for the full (partially stale) project doc.

## Repo layout — two independent git repos

`Logged/` is a git repo that orchestrates the two submodules; each subdirectory is its own repository on the `master` branch:

- `LoggedApi/` → github.com/sergio-bogaro/LoggedApi — Python/FastAPI backend
- `LoggedApp/` → github.com/sergio-bogaro/LoggedApp — React/Vite frontend

Run git commands from inside the relevant subdirectory. There is no monorepo tooling.

## Commands

Backend (workdir `LoggedApi`, with `.venv` activated):
```
uvicorn main:app --reload --port 8000      # dev server; Swagger at /docs
```

Frontend (workdir `LoggedApp`):
```
npm run dev        # Vite dev server on :5173
npm run build      # tsc -b && vite build  (the only configured typecheck path)
npx tsc -b         # typecheck-only without emitting/bundling
npm run lint       # eslint . --fix  (auto-fixes; there is no separate check-only script)
npm run preview    # serve built dist/
```

There are **no tests** in either project and no `test` script. Do not invent test commands.

## Backend conventions (`LoggedApi/`)

- **Layered**: router → service → model/schema. Never put DB queries in routers.
- **JSON is camelCase, Python is snake_case.** Every Pydantic schema uses `alias_generator=to_camel, populate_by_name=True`. Responses serialize to camelCase (`mediaId`, `coverUrl`); requests accept either form.
- **Router prefixes** (note which are NOT under `/api`): `/auth`, `/api/media`, `/api/media-logs`, `/api/favorites`, `/api/backlog`, `/api/igdb`, `/api/tmdb`.
- **Schema changes need a light migration.** No Alembic: declare new columns for existing tables in `_ADDED_COLUMNS` in `database.py`; `ensure_columns()` runs on startup and issues idempotent `ALTER TABLE`. New tables just need to be imported before `create_all`. Tables of removed features go in `_OBSOLETE_TABLES` and are dropped by `drop_obsolete_tables()` on startup.
- **Media logs carry progress** (`progress`/`progress_total`, unit implied by media type) and the media types include `music`.
- **DB is auto-created, no migrations.** `main.py` lifespan calls `Base.metadata.create_all`. `alembic` is in `requirements.txt` but unconfigured (no `alembic.ini`). Schema changes require a DB drop or manual alter; `logged.db` is gitignored.
- **Model import ordering gotcha**: `models/__init__.py` only imports `Media`, `MediaListItem`, `MediaLog`, `Tag`. `User` gets registered only because `main.py` imports routers → services → that model. When adding a model, ensure it is imported before `create_all` runs or its table won't be created.
- **Uploads**: images stored in `LoggedApi/uploads/` (gitignored except `.gitkeep`), served at `/uploads/*` via `StaticFiles` mount. Validation in `services/image_storage.py` (`.jpg/.jpeg/.png/.webp/.gif`, 10MB max).
- **Auth is intentionally simple** (single-user, personal app): plaintext passwords in `auth_service.py`, no tokens, `user_id` passed as query param. Don't "fix" these unless explicitly asked.

## Frontend conventions (`LoggedApp/`)

- **Path alias**: `@/*` → `src/*` (tsconfig + vite).
- **Directory is `src/querries/`** (double-r). Don't "correct" it to `queries` — imports would break.
- **shadcn/ui "new-york" style**. Primitives in `src/components/ui/`; custom business components in `src/components/tw/`. Aliases defined in `components.json`.
- **State split**: TanStack Query for server state, Redux Toolkit for client state (auth + UI settings only). Don't put API data in Redux.
- **External media APIs** — AniList (anime/manga), OpenLibrary (books) and iTunes (music) are called from the browser directly. **TMDB (movies) and IGDB (games) are proxied through the backend** (`/api/tmdb/*`, `/api/igdb/*`) so the API keys never reach the browser: each user can set their own keys in the app (Integrations), with an optional instance fallback in `.env` (`TMDB_API_KEY`, `IGDB_CLIENT_ID`/`IGDB_CLIENT_SECRET`). RAWG/GameBrain are **not** used despite older docs mentioning them.
- **i18n required for all user-facing strings** via `useTranslation()`. Locales: `public/locales/{en,pt-BR}/`.
- **Vite proxy**: `/anytype-api` → `http://127.0.0.1:31009/v1` (experimental, not yet wired into app code).

## Lint rules that bite (enforced in `eslint.config.js`)

- 2-space indent; double quotes; no multi-spaces.
- `import/order`: groups alphabetized with `newlines-between: always`.
- `react/jsx-curly-brace-presence`: no braces around string props/children.
- `react-hooks/exhaustive-deps` is **off**; `@typescript-eslint/no-explicit-any` is `warn`.

## Docs trustworthiness

- `INSTRUCTIONS.md` is detailed but **partially stale**: it omits the `backlog`/`favorites` routers and `media_list_item` model/service, and its "Known Issues" (duplicate router code in `media.py`) is already fixed. Trust code over that file.
- `README.md` says React 18; actual is React 19.1.

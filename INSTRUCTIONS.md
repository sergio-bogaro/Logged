# Logged - Project Instructions Archive

> **Last updated:** 2026-07-06  
> **Project:** Logged - Personal Media Tracking Application

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [Architecture](#3-architecture)
4. [Directory Structure](#4-directory-structure)
5. [Getting Started](#5-getting-started)
6. [Environment Variables](#6-environment-variables)
7. [API Reference](#7-api-reference)
8. [Frontend Routes](#8-frontend-routes)
9. [Database Schema](#9-database-schema)
10. [Development Guidelines](#10-development-guidelines)
11. [External API Integrations](#11-external-api-integrations)
12. [Build & Deployment](#12-build--deployment)
13. [Known Issues & TODOs](#13-known-issues--todos)

---

## 1. Project Overview

**Logged** is a personal media tracking application that allows a single user to track their consumption of **movies, anime, manga, games, and books** in one unified platform. Users can:

- Search for media from external APIs (TMDB, AniList, RAWG, OpenLibrary, etc.)
- Add media to their personal library
- Log progress (episodes watched, chapters read, hours played)
- Rate and review media
- Upload custom cover images
- Create custom filtered views of their library
- Track stats and visualize data with charts

The UI supports **English** and **Brazilian Portuguese**.

---

## 2. Tech Stack

### Backend (`LoggedApi/`)

| Component         | Technology          |
|-------------------|---------------------|
| Language          | Python 3.11+ (3.12 recommended) |
| Web Framework     | FastAPI 0.115.6     |
| ORM               | SQLAlchemy 2.0.36   |
| Validation        | Pydantic 2.10.4     |
| Database          | SQLite              |
| ASGI Server       | Uvicorn 0.34.0      |
| File Uploads      | python-multipart, aiofiles |
| Config            | pydantic-settings + `.env` |
| API Docs          | Auto-generated Swagger UI at `/docs` |

### Frontend (`LoggedApp/`)

| Component         | Technology          |
|-------------------|---------------------|
| Framework         | React 19.1          |
| Language          | TypeScript 5.9      |
| Build Tool        | Vite 7.1            |
| Styling           | Tailwind CSS 4.1    |
| UI Components     | shadcn/ui (New York style) |
| Routing           | React Router 7.9    |
| Client State      | Redux Toolkit 2.9   |
| Server State      | TanStack React Query 5.90 |
| Forms             | React Hook Form + Zod |
| i18n              | i18next             |
| Charts            | Chart.js            |

---

## 3. Architecture

### Backend Architecture (Router -> Service -> Model)

```
HTTP Request
    |
    v
[Router] ── Handles HTTP concerns (status codes, params, validation)
    |
    v
[Service] ── Business logic (database queries, validation, processing)
    |
    v
[Model] ── SQLAlchemy ORM models (database schema)
    |
    v
[Schema] ── Pydantic models (request/response serialization)
```

- **Routers** (`LoggedApi/routers/`): Define API endpoints and handle HTTP-layer concerns.
- **Services** (`LoggedApi/services/`): Contain all business logic and database interactions.
- **Models** (`LoggedApi/models/`): Define database tables using SQLAlchemy ORM.
- **Schemas** (`LoggedApi/schemas/`): Pydantic models for request validation and response serialization (camelCase aliases).

### Frontend Architecture

```
[Pages] ── Route-level components (protected by auth guard)
    |
    v
[Components] ── Reusable UI components (shadcn/ui + custom)
    |
    v
[Hooks/Queries] ── Data fetching (TanStack React Query)
    |
    v
[Store] ── Client state (Redux Toolkit: auth, UI preferences)
    |
    v
[API Layer] ── Direct HTTP calls to backend + external APIs
```

- **Redux Toolkit**: Manages client-side state (auth, theme, viewMode, ratingMode) with localStorage persistence.
- **TanStack React Query**: Manages server state (API data fetching, caching, stale-while-revalidate).
- **External APIs**: Called directly from the browser (not proxied through the backend).

### Authentication Model

- **No JWT/Token-based auth**: Login returns a user object stored in localStorage + Redux.
- **user_id as query parameter**: API endpoints receive `user_id` in query params, not via auth middleware.
- **WARNING**: Passwords are stored in **plaintext** (no hashing). This is a known security concern.

---

## 4. Directory Structure

```
Logged/
├── README.md
├── INSTRUCTIONS.md                 # This file
│
├── LoggedApi/                      # Backend (Python/FastAPI)
│   ├── main.py                     # FastAPI app factory & entry point
│   ├── config.py                   # Settings (pydantic-settings)
│   ├── database.py                 # SQLAlchemy engine, session, Base
│   ├── requirements.txt            # Python dependencies
│   ├── logged.db                   # SQLite database (auto-created)
│   ├── .env / .env.example         # Environment config
│   ├── models/                     # SQLAlchemy ORM models
│   │   ├── enums.py                # MediaTypeEnum, MediaStatusEnum
│   │   ├── user.py                 # User model
│   │   ├── media.py                # Media model (core entity)
│   │   ├── media_log.py            # MediaLog model (progress entries)
│   │   ├── tag.py                  # Tag model + media_tags association
│   │   └── custom_view.py          # CustomView model
│   ├── schemas/                    # Pydantic request/response schemas
│   │   ├── user.py
│   │   ├── media.py
│   │   ├── media_log.py
│   │   ├── tag.py
│   │   └── custom_view.py
│   ├── routers/                    # FastAPI route handlers
│   │   ├── auth.py                 # /auth/* endpoints
│   │   ├── media.py                # /api/media/* endpoints
│   │   ├── media_log.py            # /api/media-logs/* endpoints
│   │   └── custom_views.py         # /custom-views/* endpoints
│   ├── services/                   # Business logic layer
│   │   ├── auth_service.py         # Auth + user management
│   │   ├── media_service.py        # Media CRUD + filtering + tags
│   │   ├── media_log_service.py    # MediaLog CRUD
│   │   ├── image_storage.py        # File-based image storage
│   │   └── custom_view_service.py  # Custom views CRUD + defaults
│   └── uploads/                    # Uploaded image storage
│
└── LoggedApp/                      # Frontend (React/TypeScript/Vite)
    ├── package.json                # npm dependencies & scripts
    ├── vite.config.ts              # Vite config (plugins, proxy, aliases)
    ├── index.html                  # Vite SPA entry HTML
    ├── .env / .env.example         # Environment config
    ├── components.json             # shadcn/ui configuration
    ├── public/
    │   └── locales/                # i18n translation files
    │       ├── en/                 # English
    │       └── ptBr/               # Brazilian Portuguese
    └── src/
        ├── main.tsx                # React entry point
        ├── App.tsx                 # Root component (providers)
        ├── Router.tsx              # Route definitions
        ├── index.css               # Tailwind CSS base styles
        ├── i18n/i18n.ts           # i18next initialization
        ├── types/                  # TypeScript type definitions
        │   ├── media.ts            # MediaItem, enums
        │   ├── logged.ts           # API response/payload types
        │   ├── auth.ts             # User, AuthState types
        │   ├── settings.ts         # RatingModeEnum
        │   └── customView.ts       # CustomView types
        ├── store/                  # Redux Toolkit stores
        │   ├── settings/           # UI slice (theme, viewMode, ratingMode)
        │   └── auth/               # Auth slice (user, isAuthenticated)
        ├── querries/               # API query functions
        │   ├── auth/auth.ts        # Login/register/update API calls
        │   ├── media/
        │   │   ├── logged.ts       # Backend API: media + logs CRUD
        │   │   └── existingMedias.ts
        │   └── externalMedia/      # External API integrations
        │       ├── anilist.ts      # AniList (anime/manga)
        │       ├── mangadex.ts     # MangaDex (manga)
        │       ├── kitsu.ts        # Kitsu (anime/manga)
        │       ├── movies.ts       # TMDB (movies)
        │       ├── games.ts        # RAWG (games)
        │       ├── gamebrain.ts    # GameBrain (games)
        │       ├── books.ts        # OpenLibrary (books)
        │       └── music.ts        # iTunes (music)
        ├── layouts/
        │   └── internal.tsx        # Authenticated layout (sidebar + header)
        ├── components/
        │   ├── ProtectedRoute.tsx  # Auth guard
        │   ├── ThemeSwitcher.tsx
        │   ├── LanguageSwitcher.tsx
        │   ├── RatingSwitcher.tsx
        │   ├── ui/                 # shadcn/ui primitives
        │   └── tw/                 # Custom business components
        │       ├── header/
        │       ├── sidebar/        # AppSidebar with navigation
        │       ├── media/          # MediaCard, SearchBar, Grid, List, View
        │       ├── dialogs/        # TrackMediaDialog, MediaHistoryDialog, etc.
        │       └── generic/        # Card, Skeleton, DataExhibition, Loading
        ├── pages/
        │   ├── welcome/            # Landing page
        │   ├── auth/               # Login & Register pages
        │   ├── settings/           # User settings page
        │   └── media/
        │       ├── home/           # Dashboard (library overview + stats)
        │       ├── list/           # Filtered media list (by type)
        │       ├── search/         # External media search page
        │       └── details/        # Media detail page
        │           └── components/ # Type-specific detail subcomponents
        ├── hooks/
        │   └── use-mobile.ts      # Mobile detection hook
        ├── lib/
        │   └── utils.ts           # cn() utility (clsx + tailwind-merge)
        └── utils/                  # Utility helpers
            ├── conts.ts           # Constants (DEFAULT_STALE_TIME)
            ├── date.ts            # Date formatting helpers
            ├── string.ts          # String utilities
            ├── posterPaths.ts     # Image URL helpers
            ├── mediaText.ts       # Media type labels
            ├── mediaStore.ts      # Media store helpers
            ├── selectOptions.ts   # Select options builders
            └── generic.tsx        # Generic helpers
```

---

## 5. Getting Started

### Prerequisites

- **Python 3.11+** (3.12 recommended)
- **Node.js 18+** (with npm)
- **Git**

### Backend Setup

```bash
# Navigate to backend directory
cd LoggedApi

# Create and activate virtual environment
python -m venv .venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # macOS/Linux

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env as needed

# Run the development server
uvicorn main:app --reload --port 8000
```

- The database (`logged.db`) is auto-created on first run.
- API docs available at: http://localhost:8000/docs
- Uploaded images stored in: `LoggedApi/uploads/`

### Frontend Setup

```bash
# Navigate to frontend directory
cd LoggedApp

# Install dependencies
npm install

# Configure environment
cp .env.example .env
# Edit .env with your API keys (see Environment Variables section)

# Run the development server
npm run dev
```

- App available at: http://localhost:5173
- Vite proxies `/anytype-api` to `127.0.0.1:31009` (experimental Anytype integration)

---

## 6. Environment Variables

### Backend (`LoggedApi/.env`)

| Variable        | Description                          | Default                              |
|-----------------|--------------------------------------|--------------------------------------|
| `DATABASE_URL`  | SQLAlchemy database URL              | `sqlite:///./logged.db`             |
| `UPLOAD_DIR`    | Directory for uploaded images        | `./uploads`                          |
| `CORS_ORIGINS`  | JSON array of allowed CORS origins   | `["http://localhost:5173","http://localhost:3000"]` |

### Frontend (`LoggedApp/.env`)

| Variable              | Description                          | Required |
|-----------------------|--------------------------------------|----------|
| `VITE_API_BASE_URL`   | Backend API base URL                 | Yes (default: `http://localhost:8000`) |
| `VITE_TMDB_API_KEY`   | TMDB API key for movie search        | Yes (for movies) |
| `VITE_RAWG_API_KEY`   | RAWG API key for game search         | Yes (for RAWG) |
| `VITE_GAMEBRAIN_API_KEY` | GameBrain API key for game search  | Optional (currently used by search page) |

---

## 7. API Reference

### Base URL

```
http://localhost:8000
```

### Authentication Endpoints

| Method | Endpoint                     | Description              |
|--------|------------------------------|--------------------------|
| POST   | `/auth/register`             | Register a new user      |
| POST   | `/auth/login`                | Login (returns user obj) |
| GET    | `/auth/users/{user_id}`      | Get user details         |
| PUT    | `/auth/users/{user_id}`      | Update user              |
| DELETE | `/auth/users/{user_id}`      | Delete user              |

### Media Endpoints

| Method | Endpoint                                        | Description                          |
|--------|--------------------------------------------------|--------------------------------------|
| GET    | `/api/media/`                                    | List media (with filters)            |
| GET    | `/api/media/{media_id}`                          | Get media with logs                  |
| GET    | `/api/media/external/{external_id}`              | Check if media exists by external ID |
| GET    | `/api/media/external/{external_id}/with-logs`    | Get existing media with logs         |
| POST   | `/api/media/batch-check`                         | Batch check which external IDs exist |
| POST   | `/api/media/`                                    | Create media                         |
| PATCH  | `/api/media/{media_id}`                          | Update media                         |
| POST   | `/api/media/{media_id}/image`                    | Upload custom image                  |
| DELETE | `/api/media/{media_id}/image`                    | Remove custom image                  |
| DELETE | `/api/media/{media_id}`                          | Delete media                         |

**Query Parameters for `GET /api/media/`:**

| Parameter    | Type   | Description                    |
|-------------|--------|--------------------------------|
| `user_id`   | int    | Filter by user (required)      |
| `type`      | string | Filter by media type           |
| `status`    | string | Filter by status               |
| `search`    | string | Search in title/description    |
| `tags`      | string | Filter by tags (comma-separated) |

### Media Log Endpoints

| Method | Endpoint                              | Description         |
|--------|----------------------------------------|---------------------|
| GET    | `/api/media-logs/media/{media_id}`     | List logs for media |
| GET    | `/api/media-logs/{log_id}`             | Get single log      |
| POST   | `/api/media-logs/`                     | Create log          |
| PATCH  | `/api/media-logs/{log_id}`             | Update log          |
| DELETE | `/api/media-logs/{log_id}`             | Delete log          |

### Custom Views Endpoints

| Method | Endpoint                                         | Description             |
|--------|--------------------------------------------------|-------------------------|
| GET    | `/custom-views/user/{user_id}`                   | List user's custom views|
| POST   | `/custom-views/`                                 | Create custom view      |
| PUT    | `/custom-views/{view_id}`                        | Update custom view      |
| DELETE | `/custom-views/{view_id}`                        | Delete custom view      |
| POST   | `/custom-views/user/{user_id}/reorder`           | Reorder views           |
| POST   | `/custom-views/user/{user_id}/default`           | Create default views    |

### Utility Endpoints

| Method | Endpoint     | Description    |
|--------|--------------|----------------|
| GET    | `/`          | Health check   |
| GET    | `/health`    | Health check   |

---

## 8. Frontend Routes

| Path                            | Page                | Protected | Description                    |
|---------------------------------|---------------------|-----------|--------------------------------|
| `/`                             | WelcomePage         | No        | Landing page                   |
| `/login`                        | LoginPage           | No        | User login                     |
| `/register`                     | RegisterPage        | No        | User registration              |
| `/media/home`                   | MediaHomePage       | Yes       | Dashboard (overview + stats)   |
| `/media/list`                   | MediaListPage       | Yes       | All media list                 |
| `/media/list/:type`             | MediaListPage       | Yes       | Filtered by type               |
| `/media/:mediaType/details/:id` | MediaDetailsPage    | Yes       | Media detail page              |
| `/search`                       | MediaSearchPage     | Yes       | External media search          |
| `/settings`                     | SettingsPage        | Yes       | User settings                  |

---

## 9. Database Schema

### Entity-Relationship Diagram

```
User (1) ──< (N) Media ──< (N) Tag       (many-to-many via media_tags)
User (1) ──< (N) MediaLog
Media (1) ──< (N) MediaLog
User (1) ──< (N) CustomView
```

### Tables

#### `users`
| Column        | Type    | Description                          |
|---------------|---------|--------------------------------------|
| `id`          | INTEGER | Primary key, auto-increment          |
| `username`    | TEXT    | Unique username                      |
| `password`    | TEXT    | User password (plaintext!)           |
| `created_at`  | DATETIME| Account creation timestamp           |
| `updated_at`  | DATETIME| Last update timestamp                |
| `rating_mode` | TEXT    | Rating display mode                  |
| `view_mode`   | TEXT    | Default view mode (list/grid)        |
| `track_*`     | BOOLEAN | Toggle tracking for each media type  |

#### `media`
| Column        | Type    | Description                          |
|---------------|---------|--------------------------------------|
| `id`          | INTEGER | Primary key, auto-increment          |
| `user_id`     | INTEGER | Foreign key to users                 |
| `external_id` | TEXT    | ID from external API                 |
| `title`       | TEXT    | Media title                          |
| `description` | TEXT    | Media description                    |
| `type`        | TEXT    | MediaTypeEnum (movies/manga/anime/game/book) |
| `cover_url`   | TEXT    | Original cover URL                   |
| `image_path`  | TEXT    | Local custom image path              |
| `release_date`| DATE    | Release date                         |
| `status`      | TEXT    | MediaStatusEnum                      |
| `rating`      | REAL    | User rating                          |
| `review`      | TEXT    | User review text                     |
| `created_at`  | DATETIME| Creation timestamp                   |
| `updated_at`  | DATETIME| Last update timestamp                |

#### `media_log`
| Column        | Type    | Description                          |
|---------------|---------|--------------------------------------|
| `id`          | INTEGER | Primary key, auto-increment          |
| `user_id`     | INTEGER | Foreign key to users                 |
| `media_id`    | INTEGER | Foreign key to media                 |
| `date`        | DATE    | Log entry date                       |
| `status`      | TEXT    | Log status                           |
| `rating`      | REAL    | Log-level rating                     |
| `review`      | TEXT    | Log-level review                     |
| `start_date`  | DATE    | Start date                           |
| `end_date`    | DATE    | End date                             |
| `created_at`  | DATETIME| Creation timestamp                   |

#### `tags`
| Column  | Type    | Description                          |
|---------|---------|--------------------------------------|
| `id`    | INTEGER | Primary key, auto-increment          |
| `name`  | TEXT    | Unique tag name                      |

#### `media_tags` (Association Table)
| Column     | Type    | Description                          |
|------------|---------|--------------------------------------|
| `media_id` | INTEGER | Foreign key to media                 |
| `tag_id`   | INTEGER | Foreign key to tags                  |

#### `custom_views`
| Column              | Type    | Description                          |
|---------------------|---------|--------------------------------------|
| `id`                | INTEGER | Primary key, auto-increment          |
| `user_id`           | INTEGER | Foreign key to users                 |
| `name`              | TEXT    | View name                            |
| `description`       | TEXT    | View description                     |
| `icon`              | TEXT    | View icon                            |
| `color`             | TEXT    | View color                           |
| `order`             | INTEGER | Display order                        |
| `is_visible`        | BOOLEAN | Whether view is visible              |
| `is_pinned`         | BOOLEAN | Whether view is pinned               |
| `filters`           | JSON    | Filter configuration                 |
| `display_settings`  | JSON    | Display configuration                |
| `created_at`        | DATETIME| Creation timestamp                   |
| `updated_at`        | DATETIME| Last update timestamp                |

### Enums

#### `MediaTypeEnum`
- `movies`
- `manga`
- `anime`
- `game`
- `book`

#### `MediaStatusEnum`
- `in_progress`
- `dropped`
- `on_hold`
- `following`
- `finished`

---

## 10. Development Guidelines

### Code Style & Conventions

#### Backend (Python/FastAPI)

- **Layered architecture**: Always follow Router -> Service -> Model pattern.
- **camelCase API responses**: All Pydantic schemas use `alias_generator=to_camel` for JSON output.
- **snake_case in Python**: Variables, functions, and database columns use snake_case.
- **Services for business logic**: Never put database queries directly in routers.
- **Error handling**: Use `HTTPException` with appropriate status codes.

#### Frontend (React/TypeScript)

- **TypeScript strict mode**: All components should be properly typed.
- **shadcn/ui components**: Prefer using existing shadcn/ui components from `components/ui/`.
- **Custom components**: Place in `components/tw/` directory.
- **React Query for server state**: Use `useQuery`/`useMutation` for all API calls.
- **Redux for client state**: Only for auth and UI preferences (theme, viewMode, ratingMode).
- **i18n**: All user-facing strings must use translation keys via `useTranslation()`.
- **File naming**: Use camelCase for component files (e.g., `MediaCard.tsx`).

### Adding a New Media Type

1. **Backend**:
   - Add enum value to `MediaTypeEnum` in `models/enums.py`
   - Add type-specific fields to `media` table if needed
   - Update any type-specific logic in services

2. **Frontend**:
   - Add enum value to `MediaType` in `src/types/media.ts`
   - Create type-specific detail page components in `pages/media/details/components/`
   - Add search integration in `querries/externalMedia/`
   - Add to sidebar navigation and media type lists

### Adding a New External API Integration

1. Create a new file in `src/querries/externalMedia/` (e.g., `newApi.ts`)
2. Implement search function using the external API
3. Add React Query hook with caching and abort controller
4. Integrate into the search page (`pages/media/search/`)
5. Add API key to `.env` as `VITE_NEW_API_KEY`
6. Update `.env.example` with the new variable

### Image Upload Guidelines

- **Supported formats**: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`
- **Max size**: 10MB
- **Storage**: `LoggedApi/uploads/` directory
- **URL pattern**: `/uploads/{filename}`
- **Validation**: Enforced in `services/image_storage.py`

---

## 11. External API Integrations

The frontend directly calls multiple external media APIs:

| Media Type | API              | Auth Method           | Rate Limiting     |
|------------|------------------|-----------------------|-------------------|
| Movies     | **TMDB**         | API key (`VITE_TMDB_API_KEY`) | Standard |
| Anime      | **AniList**      | None (public)         | 90 req/min        |
| Manga      | **AniList**      | None (public)         | 90 req/min        |
| Manga (alt)| **MangaDex**     | None (public)         | Standard          |
| Anime (alt)| **Kitsu**        | None (public)         | Standard          |
| Games      | **RAWG**         | API key (`VITE_RAWG_API_KEY`) | Standard |
| Games (alt)| **GameBrain**    | API key (`VITE_GAMEBRAIN_API_KEY`) | Unknown |
| Books      | **OpenLibrary**  | None (public)         | Standard          |
| Music      | **iTunes Search**| None (public)         | Standard          |

### Caching Strategy

- External API queries use in-memory `Map` caches with configurable TTL:
  - AniList: 24 hours
  - RAWG: 24 hours
  - GameBrain: 10 hours
- Request cancellation via `AbortController`
- React Query manages cache lifecycle

---

## 12. Build & Deployment

### Development

```bash
# Backend
cd LoggedApi
uvicorn main:app --reload --port 8000

# Frontend
cd LoggedApp
npm run dev
```

### Production Build

```bash
# Frontend production build
cd LoggedApp
npm run build      # Runs: tsc -b && vite build
npm run preview    # Preview production build locally
```

### Production Deployment

```bash
# Backend
cd LoggedApi
uvicorn main:app --host 0.0.0.0 --port 8000

# Frontend
# Serve the built files from LoggedApp/dist/
# Configure reverse proxy (nginx/Apache) to serve API and static files
```

### NPM Scripts (Frontend)

| Script           | Description                          |
|------------------|--------------------------------------|
| `npm run dev`    | Start Vite development server        |
| `npm run build`  | TypeScript build + Vite production build |
| `npm run preview`| Preview production build locally     |
| `npm run lint`   | Run ESLint                           |

---

## 13. Known Issues & TODOs

### Security Concerns

1. **Plaintext passwords**: The `auth_service.py` stores and compares passwords without hashing. This should be fixed with bcrypt or similar before any production deployment.
2. **No token-based auth**: API endpoints accept `user_id` as a query parameter without verification. Anyone can impersonate any user.
3. **API keys exposed to client**: External API keys (TMDB, RAWG, GameBrain) are embedded in the frontend JavaScript bundle.

### Code Quality Issues

1. **Duplicate router code**: `LoggedApi/routers/media.py` contains nearly identical route definitions (lines 1-83 and 85-174). This appears to be a copy-paste error.
2. **Alembic not configured**: Listed in `requirements.txt` but no `alembic/` directory or `alembic.ini` exists. Database migrations are not set up.

### Planned Features (from README)

- Better structure for status/rating/review fields in media model
- Review/rating system improvements
- Statistics and data visualizations
- Anytype integration (Vite proxy configured but not implemented)

### i18n Translation Keys

- `common.*` - Common UI strings
- `themes.*` - Theme names
- `welcome.*` - Landing page text
- `media.*` - Media-related strings
- `auth.*` - Authentication strings

---

## Appendix A: API Response Format

All API responses use **camelCase** for JSON keys (e.g., `mediaId`, `coverUrl`, `createdAt`).

### Example Media Response

```json
{
  "id": 1,
  "userId": 1,
  "externalId": "12345",
  "title": "Example Movie",
  "description": "A great movie",
  "type": "movies",
  "coverUrl": "https://image.tmdb.org/t/p/w500/poster.jpg",
  "imagePath": null,
  "releaseDate": "2024-01-15",
  "status": "finished",
  "rating": 8.5,
  "review": "Loved it!",
  "createdAt": "2024-01-20T10:30:00",
  "updatedAt": "2024-02-01T15:45:00",
  "tags": [
    { "id": 1, "name": "sci-fi" },
    { "id": 2, "name": "action" }
  ],
  "logs": [
    {
      "id": 1,
      "mediaId": 1,
      "date": "2024-01-25",
      "status": "finished",
      "rating": 8.5,
      "startDate": "2024-01-20",
      "endDate": "2024-01-25"
    }
  ]
}
```

---

*This document was auto-generated by analyzing the Logged project codebase.*

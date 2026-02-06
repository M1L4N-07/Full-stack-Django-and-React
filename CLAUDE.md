# CLAUDE.md - AI Assistant Guide for Postagram

## Project Overview

Postagram is a full-stack social media application built with **Django REST Framework** (backend) and **React 18** (frontend). Users can register, create posts, comment, and like content. Based on "Full Stack Django and React" by Kolawole Mangabo (Packt Publishing).

## Repository Structure

```
├── CoreRoot/                  # Django project configuration
│   ├── settings.py            # Main settings (DB, JWT, caching, CORS)
│   ├── urls.py                # Root URL config → mounts /api/ from core.routers
│   └── wsgi.py                # WSGI entry point
├── core/                      # Django application code
│   ├── routers.py             # DRF router definitions (all API routes)
│   ├── abstract/              # Base model + serializer (UUID, timestamps)
│   ├── auth/                  # Authentication (register, login, refresh, logout)
│   ├── user/                  # User model, serializer, viewset, tests
│   ├── post/                  # Post model, serializer, viewset, tests
│   ├── comment/               # Comment model, serializer, viewset, tests
│   └── fixtures/              # Pytest fixtures (user, post, comment factories)
├── social-media-react/        # React frontend application
│   ├── src/
│   │   ├── components/        # Reusable UI components (posts/, comments/, profile/, authentication/)
│   │   ├── pages/             # Page-level components (Home, Login, Registration, Profile, etc.)
│   │   ├── hooks/             # user.actions.js — auth state management via localStorage
│   │   ├── helpers/           # axios.js (HTTP client + interceptors), test-utils.jsx
│   │   ├── routes/            # ProtectedRoute.jsx — auth guard
│   │   ├── App.js             # Root routing (react-router-dom v6, HashRouter)
│   │   └── index.js           # React DOM entry point
│   ├── package.json           # Dependencies and scripts
│   └── webpack.config.js      # Custom webpack build config
├── conftest.py                # Root pytest fixture (APIClient)
├── pytest.ini                 # Pytest settings (DJANGO_SETTINGS_MODULE)
├── .coveragerc                # Coverage config (branch=True, omits boilerplate)
├── requirements.txt           # Python dependencies
├── docker-compose.yaml        # Services: api, db (postgres), nginx, redis
├── Dockerfile                 # Python 3.10-alpine, gunicorn
├── nginx.conf                 # Reverse proxy → Django on port 8000
├── .github/workflows/         # CI/CD (backend: docker+pytest+EC2, frontend: pnpm+S3)
└── manage.py                  # Django management script
```

## Development Commands

### Backend (Django)

```bash
# Run from project root: /home/user/Full-stack-Django-and-React/

# Run tests (requires PostgreSQL and Redis, or use Docker)
pytest

# Run tests with coverage
pytest --cov=core

# Run Django dev server
python manage.py runserver

# Apply migrations
python manage.py migrate

# Create new migrations after model changes
python manage.py makemigrations

# Docker-based development (starts all services)
docker-compose up -d --build
docker-compose exec api pytest    # Run tests in container
```

### Frontend (React)

```bash
# Run from: /home/user/Full-stack-Django-and-React/social-media-react/

# Install dependencies (pnpm preferred, npm also works)
pnpm install

# Start dev server
pnpm start          # or: npm start

# Run tests
pnpm test           # or: npm test

# Production build (webpack)
pnpm build          # or: npm run build
```

## Architecture & Key Conventions

### Backend Patterns

- **Custom User Model**: `core.user.models.User` (set via `AUTH_USER_MODEL = "core_user.User"`)
- **Abstract Base Classes**: All models extend `AbstractModel` which provides:
  - `public_id` (UUID) — used as the external identifier in API responses
  - `created` / `updated` timestamps (auto-managed)
  - Custom manager with `get_object_by_public_id()` method
- **Serializer Base**: `AbstractSerializer` maps `public_id` → `id` in API responses
- **Authentication**: JWT via `djangorestframework-simplejwt` with token blacklist for logout
- **Routing**: `core/routers.py` defines all API routes using `drf-nested-routers`
  - Comments are nested under posts: `/api/post/{post_pk}/comment/`
- **Caching**: Redis cache with 5-minute TTL; cache keys invalidated on model save/delete
- **Permissions**: Custom `UserPermission` class in `core/auth/permissions.py`
  - Checks object ownership and HTTP method for fine-grained access control
- **Pagination**: `LimitOffsetPagination` with default `PAGE_SIZE = 10`
- **Database**: PostgreSQL 14 (psycopg2-binary driver)

### Frontend Patterns

- **State Management**: Auth state stored in `localStorage` (key: `auth`), no Redux store
  - `hooks/user.actions.js` provides `useUserActions()` hook for login/register/logout/edit
- **Data Fetching**: SWR library with axios; fetcher defined in `helpers/axios.js`
- **HTTP Client**: Axios with request/response interceptors for JWT token injection and auto-refresh
- **Routing**: React Router v6 with `HashRouter` — all routes use hash-based URLs
- **UI Framework**: React Bootstrap + Bootstrap 5 for styling
- **Component Tests**: Jest + React Testing Library with `jest-localstorage-mock`

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/register/` | Register new user |
| POST | `/api/auth/login/` | Login, receive JWT tokens |
| POST | `/api/auth/refresh/` | Refresh access token |
| POST | `/api/auth/logout/` | Blacklist refresh token |
| GET | `/api/user/` | List users (paginated) |
| GET/PATCH | `/api/user/{id}/` | Get/update user profile |
| GET/POST | `/api/post/` | List/create posts |
| GET/PUT/DELETE | `/api/post/{id}/` | Get/update/delete post |
| POST | `/api/post/{id}/like/` | Like a post |
| POST | `/api/post/{id}/remove_like/` | Unlike a post |
| GET/POST | `/api/post/{post_pk}/comment/` | List/create comments |
| GET/PUT/DELETE | `/api/post/{post_pk}/comment/{id}/` | Get/update/delete comment |
| POST | `/api/post/{post_pk}/comment/{id}/like/` | Like a comment |
| POST | `/api/post/{post_pk}/comment/{id}/remove_like/` | Unlike a comment |

## Testing

### Backend Testing

- **Framework**: pytest + pytest-django
- **Config**: `pytest.ini` sets `DJANGO_SETTINGS_MODULE=CoreRoot.settings`
- **Root fixture**: `conftest.py` provides `APIClient` instance
- **App fixtures**: `core/fixtures/` — `user.py`, `post.py`, `comment.py`
- **Test locations**: `core/*/tests/` directories (e.g., `core/user/tests/test_viewsets.py`)
- **Coverage**: `.coveragerc` — branch coverage, omits migrations/settings/urls/apps

### Frontend Testing

- **Framework**: Jest (via react-scripts) + React Testing Library
- **Mock data**: `src/helpers/fixtures/` — user and post mock data
- **Test locations**: `src/components/*/__tests__/` directories
- **localStorage mock**: `jest-localstorage-mock` package, `resetMocks: false` in jest config
- **ESLint**: Extends `react-app` and `react-app/jest`

## Environment Variables

### Backend (.env)

```
SECRET_KEY=              # Django secret key
DJANGO_ALLOWED_HOSTS=    # Comma-separated allowed hosts
DATABASE_NAME=           # PostgreSQL database name (default: coredb)
DATABASE_USER=           # PostgreSQL user (default: core)
DATABASE_PASSWORD=       # PostgreSQL password
DATABASE_HOST=           # Database host (default: localhost)
DATABASE_PORT=           # Database port (default: 5432)
CORS_ALLOWED_ORIGINS=    # Comma-separated CORS origins
ENV=                     # Set to "PROD" for production (disables DEBUG)
POSTGRES_USER=           # Docker compose postgres user
POSTGRES_PASSWORD=       # Docker compose postgres password
POSTGRES_DB=             # Docker compose database name
```

### Frontend (.env)

```
REACT_APP_API_URL=       # Backend API base URL (e.g., http://localhost:8000/api)
```

## Key Dependencies

### Backend (Python)

| Package | Version | Purpose |
|---------|---------|---------|
| Django | 4.0.1 | Web framework |
| djangorestframework | 3.13.1 | REST API framework |
| djangorestframework-simplejwt | 5.0.0 | JWT authentication |
| psycopg2-binary | 2.9.3 | PostgreSQL driver |
| django-cors-headers | 3.11.0 | CORS handling |
| django-filter | 21.1 | Queryset filtering |
| drf-nested-routers | 0.93.4 | Nested API routes |
| pillow | 9.3.0 | Image processing (avatars) |
| django-redis | 5.2.0 | Redis cache backend |
| gunicorn | 20.1.0 | Production WSGI server |
| pytest-django | 4.5.2 | Django test integration |

### Frontend (JavaScript)

| Package | Version | Purpose |
|---------|---------|---------|
| react | 18.0.0 | UI library |
| react-router-dom | 6.6.2 | Client-side routing |
| react-bootstrap | 2.7.0 | Bootstrap components |
| axios | 0.26.0 | HTTP client |
| axios-auth-refresh | 3.3.1 | Automatic token refresh |
| swr | 2.0.0 | Data fetching / caching |
| webpack | 5.74.0 | Module bundler |
| prettier | 2.6.2 | Code formatter |

## CI/CD

- **Backend**: `.github/workflows/ci-cd.yml` — Docker build → pytest → SSH deploy to EC2
- **Frontend**: `.github/workflows/deploy-frontend.yml` — pnpm install → test → webpack build → deploy to S3

## Important Notes for AI Assistants

1. **Custom User Model**: Always use `core.user.models.User`, never `django.contrib.auth.models.User`
2. **UUID Public IDs**: All API lookups use `public_id` (UUID), not auto-increment `id`
3. **Nested Routing**: Comments are nested under posts — viewsets receive `post_pk` from URL kwargs
4. **Cache Invalidation**: The `AbstractModel.save()` and `delete()` methods invalidate Redis cache — keep this pattern when adding new cached resources
5. **Frontend Auth**: Auth tokens stored in `localStorage` under `"auth"` key — not Redux
6. **No separate API app**: API routes are centralized in `core/routers.py`, not in individual app `urls.py` files
7. **Test Database**: Backend tests require a running PostgreSQL instance (or Docker)
8. **Media Files**: Uploaded files go to `uploads/` directory; served via nginx in production, Django in dev

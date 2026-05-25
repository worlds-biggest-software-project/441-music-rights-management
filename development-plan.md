# Music Rights Management — Phased Development Plan

> Project: 441-music-rights-management · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12+ | CWR parsing library exists in Python (weso/CWR-DataApi); pandas handles heterogeneous statement file parsing; strong async support for Celery workers; AI/ML features (anomaly detection, NLP contract analysis) are Python-native |
| API framework | FastAPI 0.115+ | Auto-generates OpenAPI 3.1 spec (standards.md requirement); Pydantic v2 integration validates JSONB schemas at the boundary; async request handling for long-running operations |
| ORM / migrations | SQLAlchemy 2.0 + Alembic | First-class JSONB column support; 2.0-style mapped classes align with Pydantic models; Alembic handles JSONB schema evolution via custom migration scripts |
| Database | PostgreSQL 16+ | Hybrid relational + JSONB model (data-model-suggestion-3); extensions: pg_trgm (catalog search), btree_gist (DATERANGE indexes), pgcrypto (column-level encryption for bank/tax data) |
| Task queue | Celery 5.4+ with Redis broker | Statement ingestion and royalty calculation are batch operations taking minutes; Celery provides task retry, monitoring (Flower), and rate limiting |
| Cache / broker | Redis 7+ | Celery broker, session cache, FX rate cache; single dependency covers multiple concerns |
| Frontend | React 19 + TypeScript + Vite | Dashboard-heavy application (catalog browse, split sheets, royalty reports); Vite for fast dev iteration; TypeScript catches API contract drift early |
| UI components | shadcn/ui + Tailwind CSS 4 | Accessible, composable components; data table and form primitives suit this domain; no runtime CSS overhead |
| HTTP client (frontend) | TanStack Query + fetch | Caching, optimistic updates, and pagination built in; auto-refetch keeps dashboards current |
| Containerisation | Docker + Docker Compose | Self-hosted deployment model per README; single `docker compose up` brings up API, worker, database, Redis |
| Testing (backend) | pytest + pytest-asyncio + factory_boy | Factory-based test data generation for complex domain objects; async test support for FastAPI endpoints |
| Testing (frontend) | Vitest + React Testing Library + Playwright | Vitest for unit/component tests; Playwright for E2E flows |
| Linter / formatter (Python) | Ruff | Replaces flake8 + isort + black; 10-100x faster; single config in pyproject.toml |
| Type checker (Python) | mypy (strict mode) | Catches JSONB access errors and None-safety issues that matter for financial calculations |
| Linter / formatter (TS) | ESLint 9 + Prettier | Standard frontend toolchain; eslint-plugin-react-hooks catches hook misuse |
| Package manager (Python) | uv | 10-100x faster than pip; lockfile-based reproducible installs; replaces pip + pip-tools + virtualenv |
| Package manager (Node) | pnpm | Disk-efficient, strict dependency resolution, workspace support |
| Key libraries | pandas, openpyxl (statement parsing); python-jose (JWT); passlib+bcrypt (passwords); httpx (async HTTP for FX/API); python-multipart (file upload); boto3 or gcsfs (document storage) |

### Project Structure

```
music-rights-management/
├── pyproject.toml
├── Dockerfile
├── Dockerfile.worker
├── docker-compose.yml
├── .env.example
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/
├── src/
│   └── mrm/
│       ├── __init__.py
│       ├── main.py                    # FastAPI app factory
│       ├── config.py                  # Pydantic Settings
│       ├── database.py                # Engine, session, base model
│       ├── celery_app.py              # Celery instance
│       ├── auth/
│       │   ├── models.py              # User SQLAlchemy model
│       │   ├── schemas.py             # Login/Register/Token Pydantic models
│       │   ├── service.py             # create_user, authenticate, issue_token
│       │   ├── router.py              # /auth/* endpoints
│       │   └── dependencies.py        # get_current_user, require_role
│       ├── catalog/
│       │   ├── models.py              # Work, Recording, Release, Party
│       │   ├── schemas.py
│       │   ├── service.py
│       │   └── router.py
│       ├── rights/
│       │   ├── models.py              # WorkShare, RecordingShare
│       │   ├── schemas.py
│       │   ├── service.py
│       │   └── router.py
│       ├── contracts/
│       │   ├── models.py              # Contract
│       │   ├── schemas.py
│       │   ├── service.py
│       │   └── router.py
│       ├── statements/
│       │   ├── models.py              # StatementSource, Batch, Line
│       │   ├── schemas.py
│       │   ├── parser.py              # Configurable CSV/Excel parser
│       │   ├── matcher.py             # Catalog matching engine
│       │   ├── service.py
│       │   ├── router.py
│       │   └── tasks.py               # Celery: parse, match, validate
│       ├── royalties/
│       │   ├── models.py              # FXRate, RoyaltyCalculation, PaymentRun
│       │   ├── schemas.py
│       │   ├── engine.py              # Core calculation logic
│       │   ├── fx.py                  # FX rate fetcher
│       │   ├── service.py
│       │   ├── router.py
│       │   └── tasks.py               # Celery: calculate, disburse
│       ├── pro/
│       │   ├── models.py              # ProRegistration
│       │   ├── schemas.py
│       │   ├── cwr.py                 # CWR 2.2 file generator
│       │   ├── service.py
│       │   └── router.py
│       ├── audit/
│       │   ├── models.py              # AuditLog
│       │   ├── service.py
│       │   └── router.py
│       └── common/
│           ├── models.py              # TimestampMixin, UUIDMixin
│           ├── schemas.py             # PaginatedResponse, ErrorResponse
│           ├── pagination.py          # Cursor/offset pagination
│           └── exceptions.py          # Domain exceptions + handlers
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── src/
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── api/                       # Generated API client from OpenAPI
│   │   ├── components/                # Shared UI components
│   │   ├── features/                  # Feature-scoped components
│   │   │   ├── catalog/
│   │   │   ├── rights/
│   │   │   ├── contracts/
│   │   │   ├── statements/
│   │   │   ├── royalties/
│   │   │   └── pro/
│   │   ├── hooks/
│   │   ├── lib/                       # Utilities
│   │   └── types/                     # Shared TypeScript types
│   └── public/
└── tests/
    ├── conftest.py                    # DB fixtures, test client, factories
    ├── factories/                     # factory_boy factories
    ├── fixtures/
    │   ├── statements/                # Sample CSV/XLSX from DSPs
    │   └── cwr/                       # Sample CWR files
    ├── unit/
    │   ├── test_parser.py
    │   ├── test_matcher.py
    │   ├── test_engine.py
    │   ├── test_cwr.py
    │   └── test_fx.py
    ├── integration/
    │   ├── test_catalog_api.py
    │   ├── test_rights_api.py
    │   ├── test_statements_api.py
    │   ├── test_royalties_api.py
    │   └── test_pro_api.py
    └── e2e/
        └── test_full_workflow.py
```

---

## Phase 1: Foundation

### Purpose
Establish the project skeleton, database connection, authentication system, and development toolchain. After this phase, a developer can run `docker compose up`, hit the health endpoint, register a user, log in, and receive a JWT. All subsequent phases build on this foundation without restructuring.

### Tasks

#### 1.1 — Project Skeleton & Toolchain

**What**: Create the project structure, dependency files, Docker configuration, and CI linting/testing setup.

**Design**:

`pyproject.toml` (key sections):
```toml
[project]
name = "music-rights-management"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.30",
    "alembic>=1.14",
    "pydantic>=2.10",
    "pydantic-settings>=2.7",
    "python-jose[cryptography]>=3.3",
    "passlib[bcrypt]>=1.7",
    "python-multipart>=0.0.18",
    "celery[redis]>=5.4",
    "redis>=5.2",
    "httpx>=0.28",
    "pandas>=2.2",
    "openpyxl>=3.1",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3",
    "pytest-asyncio>=0.24",
    "pytest-cov>=6.0",
    "factory-boy>=3.3",
    "httpx",  # TestClient transport
    "ruff>=0.8",
    "mypy>=1.13",
]

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM", "TCH"]

[tool.mypy]
strict = true
plugins = ["pydantic.mypy", "sqlalchemy.ext.mypy.plugin"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

`docker-compose.yml`:
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mrm
      POSTGRES_USER: mrm
      POSTGRES_PASSWORD: mrm_dev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mrm"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://mrm:mrm_dev@db:5432/mrm
      REDIS_URL: redis://redis:6379/0
      SECRET_KEY: dev-secret-change-in-production
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }
    volumes: ["./src:/app/src"]
    command: uvicorn mrm.main:app --host 0.0.0.0 --port 8000 --reload

  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    environment:
      DATABASE_URL: postgresql+asyncpg://mrm:mrm_dev@db:5432/mrm
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }
    command: celery -A mrm.celery_app worker --loglevel=info

volumes:
  pgdata:
```

`Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv pip install --system -e ".[dev]"
COPY . .
CMD ["uvicorn", "mrm.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`.env.example`:
```
DATABASE_URL=postgresql+asyncpg://mrm:mrm_dev@localhost:5432/mrm
REDIS_URL=redis://localhost:6379/0
SECRET_KEY=change-me-in-production
ACCESS_TOKEN_EXPIRE_MINUTES=60
CORS_ORIGINS=http://localhost:5173
```

**Testing**:
- `Unit: pyproject.toml is valid TOML and can be parsed by uv`
- `Integration: docker compose up --build succeeds; all services reach healthy state within 60s`
- `Integration: API container responds to GET /health with {"status": "ok"}`

---

#### 1.2 — Database Setup & Configuration

**What**: Configure SQLAlchemy async engine, session management, Alembic migrations, and Pydantic Settings.

**Design**:

`src/mrm/config.py`:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    secret_key: str
    access_token_expire_minutes: int = 60
    cors_origins: list[str] = ["http://localhost:5173"]
    fx_rate_source: str = "ecb"  # ecb | openexchangerates
    fx_rate_api_key: str = ""
    settlement_currency: str = "USD"
    storage_backend: str = "local"  # local | s3
    storage_path: str = "./uploads"

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

settings = Settings()  # type: ignore[call-arg]
```

`src/mrm/database.py`:
```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, MappedAsDataclass

engine = create_async_engine(settings.database_url, echo=False, pool_size=20, max_overflow=10)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session
```

`src/mrm/common/models.py` (mixins):
```python
from datetime import datetime
from uuid import UUID, uuid4
from sqlalchemy import func
from sqlalchemy.orm import Mapped, mapped_column

class UUIDMixin:
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
```

Reference data tables created in the initial migration:

```sql
CREATE TABLE territories (
    code        VARCHAR(4) PRIMARY KEY,
    name        TEXT NOT NULL,
    region      VARCHAR(50),
    cisac_code  VARCHAR(4),
    currency_code VARCHAR(3),
    is_active   BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE use_types (
    code        VARCHAR(50) PRIMARY KEY,
    name        TEXT NOT NULL,
    category    VARCHAR(30) NOT NULL,
    description TEXT,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE
);
```

Seed data: ISO 3166-1 alpha-2 country codes (~250 rows) and standard use types (~15 rows: stream, download, broadcast_radio, broadcast_tv, public_performance, sync_fee, mechanical, neighboring, user_generated_content, social_media, ringtone, fitness, gaming, print, other).

**Testing**:
- `Unit: Settings() loads from .env file with correct types`
- `Unit: Settings() raises ValidationError when DATABASE_URL is missing`
- `Integration: Alembic upgrade head creates all tables; downgrade base drops them`
- `Integration: async_session() yields a working session that can execute SELECT 1`
- `Integration: territories table seeded with ≥200 rows after migration`

---

#### 1.3 — Authentication & RBAC

**What**: User model, JWT-based authentication, role-based access control middleware.

**Design**:

User model (`src/mrm/auth/models.py`):
```python
class User(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "users"

    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    password_hash: Mapped[str | None]
    display_name: Mapped[str] = mapped_column(Text, nullable=False)
    party_id: Mapped[UUID | None] = mapped_column(ForeignKey("parties.id"))
    role: Mapped[str] = mapped_column(String(30), default="viewer")
    is_active: Mapped[bool] = mapped_column(default=True)
    mfa_enabled: Mapped[bool] = mapped_column(default=False)
    last_login_at: Mapped[datetime | None]
    preferences: Mapped[dict] = mapped_column(JSONB, default=dict)
```

Roles enum:
```python
class UserRole(str, Enum):
    ADMIN = "admin"
    MANAGER = "manager"
    ACCOUNTING = "accounting"
    PUBLISHER = "publisher"
    WRITER = "writer"
    VIEWER = "viewer"
```

Role hierarchy (each role includes permissions of roles below it):
- `admin` > `manager` > `accounting` > `publisher` > `writer` > `viewer`

Auth schemas (`src/mrm/auth/schemas.py`):
```python
class RegisterRequest(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)
    display_name: str = Field(min_length=1, max_length=255)

class LoginRequest(BaseModel):
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in: int

class UserResponse(BaseModel):
    id: UUID
    email: str
    display_name: str
    role: UserRole
    is_active: bool
    created_at: datetime
```

Auth service (`src/mrm/auth/service.py`):
```python
async def create_user(db: AsyncSession, data: RegisterRequest) -> User: ...
async def authenticate(db: AsyncSession, email: str, password: str) -> User | None: ...
def create_access_token(user_id: UUID, role: str) -> str: ...
def verify_token(token: str) -> dict: ...
```

JWT payload:
```json
{
    "sub": "user-uuid",
    "role": "manager",
    "exp": 1717000000,
    "iat": 1716996400
}
```

Dependencies (`src/mrm/auth/dependencies.py`):
```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User: ...

def require_role(minimum_role: UserRole) -> Callable:
    """Returns a dependency that enforces minimum role level."""
    ...
```

API endpoints:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /auth/register | RegisterRequest | UserResponse | None |
| POST | /auth/login | LoginRequest | TokenResponse | None |
| GET | /auth/me | — | UserResponse | JWT |

FastAPI app factory (`src/mrm/main.py`):
```python
def create_app() -> FastAPI:
    app = FastAPI(
        title="Music Rights Management",
        version="0.1.0",
        docs_url="/docs",
        openapi_url="/openapi.json",
    )
    app.add_middleware(CORSMiddleware, allow_origins=settings.cors_origins, ...)
    app.include_router(auth_router, prefix="/auth", tags=["auth"])
    app.include_router(catalog_router, prefix="/catalog", tags=["catalog"])
    # ... additional routers added in later phases
    return app

app = create_app()
```

Error handling (`src/mrm/common/exceptions.py`):
```python
class NotFoundError(Exception): ...
class ConflictError(Exception): ...
class ForbiddenError(Exception): ...

# Registered as FastAPI exception handlers returning RFC 7807 Problem JSON:
# {"type": "not_found", "title": "Not Found", "detail": "Work xyz not found", "status": 404}
```

**Testing**:
- `Unit: create_access_token → token decodes with correct sub, role, and exp`
- `Unit: verify_token with expired token → raises ExpiredTokenError`
- `Unit: verify_token with tampered token → raises InvalidTokenError`
- `Unit: password hashing → verify_password(hash, "correct") returns True`
- `Unit: password hashing → verify_password(hash, "wrong") returns False`
- `Integration: POST /auth/register with valid data → 201, user in DB`
- `Integration: POST /auth/register with duplicate email → 409 Conflict`
- `Integration: POST /auth/login with correct credentials → 200, valid JWT`
- `Integration: POST /auth/login with wrong password → 401 Unauthorized`
- `Integration: GET /auth/me with valid JWT → 200, correct user data`
- `Integration: GET /auth/me without token → 401`
- `Integration: require_role(MANAGER) allows admin user → passes`
- `Integration: require_role(MANAGER) blocks viewer user → 403 Forbidden`

---

## Phase 2: Catalog & Parties

### Purpose
Implement CRUD for the core domain entities: parties (rights holders, publishers, labels, PROs), works (compositions), recordings (masters), releases, and the linking relationships between them. After this phase, a user can populate a full catalog with proper identifiers (ISWC, ISRC, IPI, UPC), link works to recordings, and search the catalog. This is the foundation all rights, statements, and royalty features depend on.

### Tasks

#### 2.1 — Party Management

**What**: Full CRUD for parties (songwriters, publishers, labels, PROs, DSPs) with identifier storage and search.

**Design**:

Party model (hybrid relational + JSONB, per data-model-suggestion-3):
```python
class Party(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "parties"

    party_type: Mapped[str] = mapped_column(String(30), nullable=False)
    legal_name: Mapped[str] = mapped_column(Text, nullable=False)
    display_name: Mapped[str | None] = mapped_column(Text)
    ipi_name_number: Mapped[str | None] = mapped_column(String(11), unique=True)
    country_code: Mapped[str | None] = mapped_column(String(2))
    status: Mapped[str] = mapped_column(String(20), default="active")
    pro_affiliation_id: Mapped[UUID | None] = mapped_column(ForeignKey("parties.id"))
    identifiers: Mapped[dict] = mapped_column(JSONB, default=dict)
    contact_info: Mapped[dict] = mapped_column(JSONB, default=dict)
```

`identifiers` JSONB schema:
```json
{
    "ipi_base_number": "I-000000123-4",
    "ipn": "12345678",
    "isni": "0000 0001 2345 6789",
    "mbid": "a1b2c3d4-e5f6-...",
    "pro_member_ids": { "ASCAP": "ASC-123456", "BMI": "BMI-789012" },
    "tax_ids": { "US": {"type": "SSN", "value": "encrypted:..."} }
}
```

`contact_info` JSONB schema:
```json
{
    "email": "artist@example.com",
    "phone": "+1-555-0123",
    "address": { "line_1": "123 Music Row", "city": "Nashville", "state": "TN", "postal_code": "37203", "country": "US" },
    "bank_accounts": [
        {
            "id": "uuid",
            "account_name": "Artist A",
            "currency": "USD",
            "bank_name": "First Bank",
            "routing_number": "encrypted:...",
            "account_number": "encrypted:...",
            "payment_method": "bank_transfer",
            "is_primary": true,
            "is_verified": false
        }
    ],
    "payment_preferences": {
        "preferred_method": "bank_transfer",
        "minimum_payout": 50.00,
        "stripe_connect_id": null,
        "paypal_email": null
    }
}
```

Party type enum:
```python
class PartyType(str, Enum):
    SONGWRITER = "songwriter"
    COMPOSER = "composer"
    LYRICIST = "lyricist"
    ARRANGER = "arranger"
    PERFORMER = "performer"
    PUBLISHER = "publisher"
    SUB_PUBLISHER = "sub_publisher"
    ADMINISTRATOR = "administrator"
    LABEL = "label"
    DISTRIBUTOR = "distributor"
    PRO = "pro"
    DSP = "dsp"
```

Schemas:
```python
class PartyCreate(BaseModel):
    party_type: PartyType
    legal_name: str = Field(min_length=1, max_length=500)
    display_name: str | None = None
    ipi_name_number: str | None = Field(None, pattern=r"^[0-9]{9,11}$")
    country_code: str | None = Field(None, pattern=r"^[A-Z]{2}$")
    pro_affiliation_id: UUID | None = None
    identifiers: dict = Field(default_factory=dict)
    contact_info: dict = Field(default_factory=dict)

class PartyUpdate(BaseModel):
    legal_name: str | None = None
    display_name: str | None = None
    ipi_name_number: str | None = None
    country_code: str | None = None
    status: str | None = None
    pro_affiliation_id: UUID | None = None
    identifiers: dict | None = None
    contact_info: dict | None = None

class PartyResponse(BaseModel):
    id: UUID
    party_type: PartyType
    legal_name: str
    display_name: str | None
    ipi_name_number: str | None
    country_code: str | None
    status: str
    pro_affiliation_id: UUID | None
    identifiers: dict
    contact_info: dict
    created_at: datetime
    updated_at: datetime

class PartyListParams(BaseModel):
    party_type: PartyType | None = None
    status: str | None = None
    search: str | None = None  # pg_trgm similarity on legal_name
    offset: int = 0
    limit: int = Field(default=50, le=200)
```

API endpoints:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /catalog/parties | PartyCreate | PartyResponse | manager+ |
| GET | /catalog/parties | PartyListParams (query) | PaginatedResponse[PartyResponse] | viewer+ |
| GET | /catalog/parties/{id} | — | PartyResponse | viewer+ |
| PATCH | /catalog/parties/{id} | PartyUpdate | PartyResponse | manager+ |
| DELETE | /catalog/parties/{id} | — | 204 | admin |

Service functions:
```python
async def create_party(db: AsyncSession, data: PartyCreate, created_by: UUID) -> Party: ...
async def get_party(db: AsyncSession, party_id: UUID) -> Party: ...
async def list_parties(db: AsyncSession, params: PartyListParams) -> tuple[list[Party], int]: ...
async def update_party(db: AsyncSession, party_id: UUID, data: PartyUpdate) -> Party: ...
async def delete_party(db: AsyncSession, party_id: UUID) -> None: ...
```

Search: `pg_trgm` GIN index on `legal_name` enables `%` similarity search with `similarity(legal_name, :query) > 0.3` threshold.

IPI validation: IPI Name Numbers are 9-11 digit numeric strings assigned by CISAC. Validate format on create/update.

**Testing**:
- `Unit: PartyCreate with valid IPI "00123456789" → passes validation`
- `Unit: PartyCreate with invalid IPI "ABC" → ValidationError`
- `Unit: PartyCreate with invalid country_code "USA" → ValidationError`
- `Integration: POST /catalog/parties → 201, party persisted with correct fields`
- `Integration: POST /catalog/parties with duplicate IPI → 409 Conflict`
- `Integration: GET /catalog/parties?search=songwriter → returns matching parties by name similarity`
- `Integration: GET /catalog/parties?party_type=publisher → filters by type`
- `Integration: PATCH /catalog/parties/{id} with new identifiers → merges JSONB correctly`
- `Integration: DELETE /catalog/parties/{id} that has linked shares → 409 (referential integrity)`
- `Integration: viewer role can GET but not POST → 403`

---

#### 2.2 — Work (Composition) Management

**What**: Full CRUD for musical works with ISWC validation, alternate titles, AI contribution tracking, and metadata.

**Design**:

Work model:
```python
class Work(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "works"

    title: Mapped[str] = mapped_column(Text, nullable=False)
    iswc: Mapped[str | None] = mapped_column(String(15), unique=True)
    work_type: Mapped[str] = mapped_column(String(50), default="MusicalWork")
    status: Mapped[str] = mapped_column(String(30), default="draft")
    created_by: Mapped[UUID | None] = mapped_column(ForeignKey("users.id"))
    metadata_: Mapped[dict] = mapped_column("metadata", JSONB, default=dict)
    cwr_data: Mapped[dict | None] = mapped_column(JSONB)
```

`metadata` JSONB schema:
```json
{
    "alternate_titles": ["Title Variant 1"],
    "language_code": "en",
    "duration_seconds": 214,
    "genre": "Pop",
    "sub_genre": "Synth Pop",
    "ai_contribution": { "is_ai": false, "ai_percentage": 0, "ai_tool": null, "human_elements": [] },
    "creation_date": "2026-01-10"
}
```

Work status lifecycle: `draft` → `registered` → `withdrawn` (or `disputed` from any active state).

ISWC validation (ISO 15707): format `T-XXXXXXXXX-C` where X is a digit and C is a check digit.
```python
def validate_iswc(value: str) -> str:
    if not re.match(r"^T-\d{9}-\d$", value):
        raise ValueError("ISWC must match T-XXXXXXXXX-C format")
    return value
```

API endpoints:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /catalog/works | WorkCreate | WorkResponse | publisher+ |
| GET | /catalog/works | WorkListParams (query) | PaginatedResponse[WorkResponse] | viewer+ |
| GET | /catalog/works/{id} | — | WorkDetailResponse (includes linked recordings, shares) | viewer+ |
| PATCH | /catalog/works/{id} | WorkUpdate | WorkResponse | publisher+ |
| DELETE | /catalog/works/{id} | — | 204 | admin |

**Testing**:
- `Unit: validate_iswc("T-345246800-1") → passes`
- `Unit: validate_iswc("X-345246800-1") → raises ValueError`
- `Unit: validate_iswc("T-34524680-1") → raises ValueError (9 digits required)`
- `Integration: POST /catalog/works with valid ISWC → 201`
- `Integration: POST /catalog/works with duplicate ISWC → 409`
- `Integration: GET /catalog/works?search=midnight → returns works with matching titles`
- `Integration: GET /catalog/works/{id} → includes linked recordings and active shares`
- `Integration: PATCH /catalog/works/{id} metadata with AI contribution → updates JSONB correctly`

---

#### 2.3 — Recording (Master) Management

**What**: Full CRUD for sound recordings with ISRC validation, artist/label linking, and audio metadata.

**Design**:

Recording model:
```python
class Recording(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "recordings"

    title: Mapped[str] = mapped_column(Text, nullable=False)
    isrc: Mapped[str | None] = mapped_column(String(12), unique=True)
    recording_type: Mapped[str] = mapped_column(String(30), default="SoundRecording")
    primary_artist_id: Mapped[UUID | None] = mapped_column(ForeignKey("parties.id"))
    label_id: Mapped[UUID | None] = mapped_column(ForeignKey("parties.id"))
    status: Mapped[str] = mapped_column(String(30), default="active")
    created_by: Mapped[UUID | None] = mapped_column(ForeignKey("users.id"))
    metadata_: Mapped[dict] = mapped_column("metadata", JSONB, default=dict)
    ddex_ern_data: Mapped[dict | None] = mapped_column(JSONB)
```

ISRC validation (ISO 3901): 12-character alphanumeric `CC-XXX-YY-NNNNN` stored without hyphens as `CCXXXYYNNNNN`.
```python
def validate_isrc(value: str) -> str:
    clean = value.replace("-", "")
    if not re.match(r"^[A-Z]{2}[A-Z0-9]{3}\d{7}$", clean):
        raise ValueError("ISRC must be 12 chars: CC-XXX-YY-NNNNN")
    return clean
```

Recording types: `SoundRecording`, `MusicVideo`, `Remix`, `LiveRecording`, `Remaster`.

API endpoints mirror Work pattern: POST/GET/GET{id}/PATCH/DELETE under `/catalog/recordings`.

**Testing**:
- `Unit: validate_isrc("US-RC1-23-45678") → "USRC12345678"`
- `Unit: validate_isrc("123") → raises ValueError`
- `Integration: POST /catalog/recordings with primary_artist_id → artist party linked`
- `Integration: GET /catalog/recordings?search=midnight → returns matching recordings`
- `Integration: GET /catalog/recordings/{id} → includes linked works and artist details`

---

#### 2.4 — Work-Recording Links, Releases & Catalog Search

**What**: Linking works to recordings (many-to-many), release/track management, and unified catalog search.

**Design**:

WorkRecording model:
```python
class WorkRecording(Base, UUIDMixin):
    __tablename__ = "work_recordings"

    work_id: Mapped[UUID] = mapped_column(ForeignKey("works.id", ondelete="CASCADE"))
    recording_id: Mapped[UUID] = mapped_column(ForeignKey("recordings.id", ondelete="CASCADE"))
    relationship: Mapped[str] = mapped_column(String(30), default="performance")
    is_primary: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    __table_args__ = (UniqueConstraint("work_id", "recording_id"),)
```

Relationship types: `performance`, `cover`, `sample`, `interpolation`, `remix`, `medley`.

Release and ReleaseTrack models:
```python
class Release(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "releases"

    title: Mapped[str] = mapped_column(Text, nullable=False)
    upc: Mapped[str | None] = mapped_column(String(14), unique=True)
    release_type: Mapped[str] = mapped_column(String(30), default="single")
    release_date: Mapped[date | None]
    label_id: Mapped[UUID | None] = mapped_column(ForeignKey("parties.id"))
    status: Mapped[str] = mapped_column(String(30), default="active")
    metadata_: Mapped[dict] = mapped_column("metadata", JSONB, default=dict)

class ReleaseTrack(Base, UUIDMixin):
    __tablename__ = "release_tracks"

    release_id: Mapped[UUID] = mapped_column(ForeignKey("releases.id", ondelete="CASCADE"))
    recording_id: Mapped[UUID] = mapped_column(ForeignKey("recordings.id"))
    disc_number: Mapped[int] = mapped_column(SmallInteger, default=1)
    track_number: Mapped[int] = mapped_column(SmallInteger)

    __table_args__ = (UniqueConstraint("release_id", "disc_number", "track_number"),)
```

Link endpoints:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /catalog/works/{work_id}/recordings | {recording_id, relationship, is_primary} | WorkRecordingResponse | publisher+ |
| DELETE | /catalog/works/{work_id}/recordings/{recording_id} | — | 204 | publisher+ |
| GET | /catalog/works/{work_id}/recordings | — | list[RecordingResponse] | viewer+ |
| GET | /catalog/recordings/{recording_id}/works | — | list[WorkResponse] | viewer+ |

Release endpoints: standard CRUD under `/catalog/releases`, plus track management under `/catalog/releases/{id}/tracks`.

Catalog search endpoint:
| Method | Path | Query | Response | Auth |
|--------|------|-------|----------|------|
| GET | /catalog/search | q (text), type (work\|recording\|party\|release), limit, offset | PaginatedResponse[CatalogSearchResult] | viewer+ |

Search implementation uses `pg_trgm` similarity across work titles, recording titles, party names, ISWCs, ISRCs, and UPCs.

**Testing**:
- `Integration: POST /catalog/works/{id}/recordings links work and recording`
- `Integration: POST with non-existent recording_id → 404`
- `Integration: POST duplicate link → 409`
- `Integration: DELETE link → 204, link removed`
- `Integration: GET /catalog/search?q=midnight&type=work → returns matching works`
- `Integration: GET /catalog/search?q=USRC12345678 → returns recording by ISRC`
- `Integration: Release with tracks → GET /catalog/releases/{id} includes track listing with recordings`
- `Fixture: Create work + recording + link → GET /catalog/works/{id} returns linked recordings`

---

## Phase 3: Rights & Split Management

### Purpose
Implement version-controlled ownership splits for both compositions (work shares) and master recordings (recording shares), with territory-specific overrides stored as JSONB. After this phase, a publisher can record that "Songwriter A owns 25% performance rights in the US, 30% in Germany via sub-publisher, effective from January 2026" — and the system preserves the full history when splits change. This is the core of rights administration.

### Tasks

#### 3.1 — Work Share Management

**What**: CRUD for composition ownership splits with per-right-type percentages, versioning, and territory configuration.

**Design**:

WorkShare model (hybrid relational + JSONB, per data-model-suggestion-3):
```python
class WorkShare(Base, UUIDMixin):
    __tablename__ = "work_shares"

    work_id: Mapped[UUID] = mapped_column(ForeignKey("works.id", ondelete="CASCADE"))
    party_id: Mapped[UUID] = mapped_column(ForeignKey("parties.id"))
    role: Mapped[str] = mapped_column(String(30), nullable=False)

    performance_share: Mapped[Decimal] = mapped_column(Numeric(7, 4), default=0)
    mechanical_share: Mapped[Decimal] = mapped_column(Numeric(7, 4), default=0)
    sync_share: Mapped[Decimal] = mapped_column(Numeric(7, 4), default=0)
    print_share: Mapped[Decimal] = mapped_column(Numeric(7, 4), default=0)
    controlled: Mapped[bool] = mapped_column(default=False)
    publisher_party_id: Mapped[UUID | None] = mapped_column(ForeignKey("parties.id"))

    version: Mapped[int] = mapped_column(default=1)
    effective_from: Mapped[date] = mapped_column(default=date.today)
    effective_to: Mapped[date | None]
    superseded_by: Mapped[UUID | None] = mapped_column(ForeignKey("work_shares.id"))

    territory_config: Mapped[dict] = mapped_column(JSONB, default=dict)
    cwr_share_data: Mapped[dict | None] = mapped_column(JSONB)
    notes: Mapped[str | None] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    created_by: Mapped[UUID | None] = mapped_column(ForeignKey("users.id"))
```

`territory_config` JSONB schema:
```json
{
    "default_territories": "worldwide",
    "excluded_territories": ["CN", "RU"],
    "territory_overrides": {
        "US": {
            "performance_share": 30.00,
            "mechanical_share": 30.00,
            "collection_society": "ASCAP",
            "society_work_id": "ASC-890123"
        },
        "DE": {
            "performance_share": 20.00,
            "mechanical_share": 20.00,
            "collection_society": "GEMA",
            "sub_publisher_id": "subpub-de-...",
            "sub_publisher_share": 15.00
        }
    },
    "use_type_overrides": {
        "sync": { "share": 50.00 }
    }
}
```

Writer roles: `composer`, `lyricist`, `arranger`, `author`, `adapter`, `translator`.
Publisher roles: `original_publisher`, `sub_publisher`, `administrator`.

Share versioning logic:
1. Creating a new share: insert with `version=1`, `effective_from=today`, `effective_to=NULL`.
2. Revising a share: set `effective_to` and `superseded_by` on the current version; insert new row with `version=N+1`.
3. Querying active shares: `WHERE effective_to IS NULL AND superseded_by IS NULL`.

Schemas:
```python
class WorkShareCreate(BaseModel):
    party_id: UUID
    role: str = Field(pattern=r"^(composer|lyricist|arranger|author|adapter|translator|original_publisher|sub_publisher|administrator)$")
    performance_share: Decimal = Field(ge=0, le=100, decimal_places=4)
    mechanical_share: Decimal = Field(ge=0, le=100, decimal_places=4)
    sync_share: Decimal = Field(ge=0, le=100, decimal_places=4)
    print_share: Decimal = Field(ge=0, le=100, decimal_places=4)
    controlled: bool = False
    publisher_party_id: UUID | None = None
    effective_from: date = Field(default_factory=date.today)
    territory_config: dict = Field(default_factory=dict)
    notes: str | None = None

class WorkShareRevise(BaseModel):
    performance_share: Decimal | None = Field(None, ge=0, le=100)
    mechanical_share: Decimal | None = Field(None, ge=0, le=100)
    sync_share: Decimal | None = Field(None, ge=0, le=100)
    print_share: Decimal | None = Field(None, ge=0, le=100)
    territory_config: dict | None = None
    effective_from: date = Field(default_factory=date.today)
    reason: str

class WorkShareResponse(BaseModel):
    id: UUID
    work_id: UUID
    party_id: UUID
    party_name: str  # denormalized from Party
    role: str
    performance_share: Decimal
    mechanical_share: Decimal
    sync_share: Decimal
    print_share: Decimal
    controlled: bool
    publisher_party_id: UUID | None
    version: int
    effective_from: date
    effective_to: date | None
    territory_config: dict
    notes: str | None
    created_at: datetime
```

API endpoints:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /rights/works/{work_id}/shares | WorkShareCreate | WorkShareResponse | publisher+ |
| GET | /rights/works/{work_id}/shares | ?active_only=true&as_of=date | list[WorkShareResponse] | viewer+ |
| GET | /rights/works/{work_id}/shares/{share_id} | — | WorkShareResponse | viewer+ |
| PUT | /rights/works/{work_id}/shares/{share_id}/revise | WorkShareRevise | WorkShareResponse (new version) | publisher+ |
| GET | /rights/works/{work_id}/shares/{share_id}/history | — | list[WorkShareResponse] (all versions) | viewer+ |
| GET | /rights/works/{work_id}/share-totals | — | ShareTotalsResponse | viewer+ |

Share total validation function (called on create and revise, warning-only — some industry conventions allow >100% for writer/publisher splits):
```python
async def check_share_totals(db: AsyncSession, work_id: UUID) -> ShareTotals:
    """Returns total active shares per right type. Warns if >100% or <100%."""
    ...
```

`get_effective_share` helper for royalty calculation:
```python
def get_effective_share(
    share: WorkShare,
    territory_code: str,
    rights_type: str,
) -> Decimal:
    """Returns the effective share percentage, checking territory overrides first."""
    overrides = share.territory_config.get("territory_overrides", {})
    territory = overrides.get(territory_code, {})
    override_key = f"{rights_type}_share"
    if override_key in territory:
        return Decimal(str(territory[override_key]))
    excluded = share.territory_config.get("excluded_territories", [])
    if territory_code in excluded:
        return Decimal("0")
    return getattr(share, f"{rights_type}_share")
```

**Testing**:
- `Unit: get_effective_share with territory override → returns override value`
- `Unit: get_effective_share with excluded territory → returns 0`
- `Unit: get_effective_share with no override → returns base share`
- `Unit: WorkShareCreate with share > 100 → ValidationError`
- `Integration: POST share → 201, share persisted with correct version=1`
- `Integration: PUT revise → original gets effective_to+superseded_by; new version created with version=2`
- `Integration: GET shares?active_only=true → returns only current versions`
- `Integration: GET shares?as_of=2025-01-01 → returns shares effective at that date`
- `Integration: GET share-totals → returns sum per right type with warning if != 100%`
- `Integration: GET history → returns all versions ordered by version number`

---

#### 3.2 — Recording Share Management

**What**: CRUD for master recording ownership splits with the same versioning pattern as work shares.

**Design**:

RecordingShare model:
```python
class RecordingShare(Base, UUIDMixin):
    __tablename__ = "recording_shares"

    recording_id: Mapped[UUID] = mapped_column(ForeignKey("recordings.id", ondelete="CASCADE"))
    party_id: Mapped[UUID] = mapped_column(ForeignKey("parties.id"))
    role: Mapped[str] = mapped_column(String(30), nullable=False)

    master_share: Mapped[Decimal] = mapped_column(Numeric(7, 4), default=0)
    neighboring_share: Mapped[Decimal] = mapped_column(Numeric(7, 4), default=0)

    version: Mapped[int] = mapped_column(default=1)
    effective_from: Mapped[date] = mapped_column(default=date.today)
    effective_to: Mapped[date | None]
    superseded_by: Mapped[UUID | None] = mapped_column(ForeignKey("recording_shares.id"))

    territory_config: Mapped[dict] = mapped_column(JSONB, default=dict)
    notes: Mapped[str | None] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    created_by: Mapped[UUID | None] = mapped_column(ForeignKey("users.id"))
```

Recording roles: `featured_artist`, `producer`, `label`, `mixer`, `remixer`, `session_musician`.

API endpoints mirror work shares under `/rights/recordings/{recording_id}/shares`.

**Testing**:
- `Integration: POST recording share → 201, share with master_share and neighboring_share`
- `Integration: PUT revise → versioning works identically to work shares`
- `Integration: GET share-totals → validates master_share totals`

---

#### 3.3 — Share Validation & Territory Resolution

**What**: Business rule validation for share totals, territory exclusion logic, and a consolidated "rights summary" endpoint.

**Design**:

ShareTotalsResponse:
```python
class ShareTotalsResponse(BaseModel):
    work_id: UUID
    performance_total: Decimal
    mechanical_total: Decimal
    sync_total: Decimal
    print_total: Decimal
    share_count: int
    warnings: list[str]  # e.g., "Performance shares total 105.00%, expected 100%"
```

Rights summary endpoint — given a work/recording and territory, returns the resolved shares for every party:
| Method | Path | Query | Response | Auth |
|--------|------|-------|----------|------|
| GET | /rights/works/{work_id}/resolved | territory=DE&rights_type=performance&as_of=2026-03-15 | list[ResolvedShareResponse] | viewer+ |

```python
class ResolvedShareResponse(BaseModel):
    party_id: UUID
    party_name: str
    role: str
    base_share: Decimal
    territory_override: Decimal | None
    effective_share: Decimal
    territory_code: str
    rights_type: str
    as_of: date
```

This endpoint is critical for the royalty calculation engine (Phase 6) and for the PRO registration export (Phase 7).

**Testing**:
- `Unit: resolve shares for US territory with no overrides → base shares returned`
- `Unit: resolve shares for DE with override → override values used`
- `Unit: resolve shares for excluded territory → all shares 0`
- `Integration: GET resolved for work with 3 parties → returns 3 entries with correct effective shares`
- `Integration: GET resolved with as_of date before share creation → empty result`
- `Fixture: Work with parties having overlapping territory overrides → correct resolution per party`

---

## Phase 4: Contract Management

### Purpose
Implement storage and management of recording agreements, co-publishing deals, sub-publishing agreements, sync licences, and other contract types. Contracts use JSONB for type-specific terms (advances, royalty rates, option periods, reversion clauses), keeping the schema flexible across the wide variety of music industry agreements. After this phase, a publisher can store contracts, track option deadlines, and receive alerts for upcoming reversions.

### Tasks

#### 4.1 — Contract CRUD with JSONB Terms

**What**: Contract model with relational fields for universal attributes and JSONB for type-specific terms.

**Design**:

Contract model:
```python
class Contract(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "contracts"

    contract_type: Mapped[str] = mapped_column(String(50), nullable=False)
    title: Mapped[str] = mapped_column(Text, nullable=False)
    contract_number: Mapped[str | None] = mapped_column(String(50))
    status: Mapped[str] = mapped_column(String(30), default="draft")
    effective_date: Mapped[date | None]
    expiry_date: Mapped[date | None]
    created_by: Mapped[UUID | None] = mapped_column(ForeignKey("users.id"))

    parties: Mapped[list[dict]] = mapped_column(JSONB, default=list)
    territories: Mapped[dict] = mapped_column(JSONB, default=lambda: {"scope": "worldwide"})
    covered_works: Mapped[dict] = mapped_column(JSONB, default=lambda: {"scope": "specific", "items": []})
    terms: Mapped[dict] = mapped_column(JSONB, default=dict)
    ai_analysis: Mapped[dict | None] = mapped_column(JSONB)
    documents: Mapped[list[dict]] = mapped_column(JSONB, default=list)
```

Contract types: `recording_agreement`, `co_publishing`, `sub_publishing`, `admin_agreement`, `sync_licence`, `master_use_licence`, `distribution_agreement`, `songwriter_agreement`, `producer_agreement`.

Contract status lifecycle: `draft` → `active` → `expired` | `terminated` (or `disputed` from `active`).

`terms` JSONB varies by contract_type. Example for `co_publishing`:
```json
{
    "advance": { "amount": 50000.00, "currency": "USD", "recoupable": true, "recouped_to_date": 12500.00, "recoupment_rate": 100 },
    "royalty_rates": {
        "mechanical": { "writer_share": 50.00, "publisher_share": 50.00 },
        "performance": { "writer_share": 50.00, "publisher_share": 50.00 },
        "sync": { "split": 50.00, "minimum_fee": 5000.00, "approval_required": true }
    },
    "option_periods": [
        { "period_number": 1, "start_date": "2026-01-01", "end_date": "2027-12-31", "exercise_deadline": "2027-09-30", "status": "exercised", "advance": 25000.00 },
        { "period_number": 2, "start_date": "2028-01-01", "end_date": "2029-12-31", "exercise_deadline": "2029-09-30", "status": "pending", "advance": 30000.00 }
    ],
    "reversion_clauses": [
        { "trigger_type": "out_of_print", "condition": "No commercial release for 12 consecutive months", "reversion_scope": "full", "notice_period_days": 90, "status": "pending" }
    ],
    "delivery_commitment": { "minimum_works": 12, "delivered_works": 5, "period": "per_option_period" },
    "auto_renew": false,
    "governing_law": "New York, USA"
}
```

Application-level JSONB validation per contract type:
```python
def validate_contract_terms(contract_type: str, terms: dict) -> list[str]:
    """Returns list of validation errors. Empty list = valid."""
    errors = []
    if contract_type == "co_publishing":
        if "royalty_rates" not in terms:
            errors.append("co_publishing contracts require 'royalty_rates'")
    elif contract_type == "sync_licence":
        if "licence_fee" not in terms:
            errors.append("sync_licence contracts require 'licence_fee'")
        if "usage" not in terms:
            errors.append("sync_licence contracts require 'usage'")
    return errors
```

API endpoints:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /contracts | ContractCreate | ContractResponse | manager+ |
| GET | /contracts | ContractListParams | PaginatedResponse[ContractResponse] | viewer+ |
| GET | /contracts/{id} | — | ContractDetailResponse | viewer+ |
| PATCH | /contracts/{id} | ContractUpdate | ContractResponse | manager+ |
| DELETE | /contracts/{id} | — | 204 | admin |
| POST | /contracts/{id}/documents | multipart file upload | DocumentResponse | manager+ |

**Testing**:
- `Unit: validate_contract_terms("co_publishing", {}) → ["co_publishing contracts require 'royalty_rates'"]`
- `Unit: validate_contract_terms("sync_licence", valid_terms) → []`
- `Integration: POST contract with valid co-pub terms → 201`
- `Integration: POST contract with invalid terms → 422 with validation errors`
- `Integration: GET /contracts?contract_type=sync_licence → filtered list`
- `Integration: PATCH terms JSONB → merges correctly without overwriting existing nested keys`

---

#### 4.2 — Contract-Party and Contract-Work Associations

**What**: Link contracts to their parties and covered works/recordings, with proper validation.

**Design**:

The `parties` and `covered_works` JSONB fields on the Contract model store these associations. Service functions validate that referenced party_ids and work_ids exist:

```python
async def validate_contract_references(db: AsyncSession, contract: ContractCreate) -> list[str]:
    errors = []
    for p in contract.parties:
        if not await party_exists(db, p["party_id"]):
            errors.append(f"Party {p['party_id']} not found")
    if contract.covered_works.get("scope") == "specific":
        for item in contract.covered_works.get("items", []):
            if "work_id" in item and not await work_exists(db, item["work_id"]):
                errors.append(f"Work {item['work_id']} not found")
    return errors
```

Convenience endpoints for adding/removing parties and works without replacing the entire JSONB:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /contracts/{id}/parties | {party_id, role} | ContractResponse | manager+ |
| DELETE | /contracts/{id}/parties/{party_id} | — | ContractResponse | manager+ |
| POST | /contracts/{id}/works | {work_id} | ContractResponse | manager+ |
| DELETE | /contracts/{id}/works/{work_id} | — | ContractResponse | manager+ |

**Testing**:
- `Integration: POST party to contract → party appears in contract.parties JSONB`
- `Integration: POST non-existent party → 404`
- `Integration: DELETE party from contract → removed from JSONB`
- `Integration: POST work to contract → work appears in covered_works.items`
- `Integration: GET /contracts?party_id=X → returns contracts involving that party (JSONB containment query)`

---

#### 4.3 — Option Periods, Reversion Triggers & Alert System

**What**: Extract and monitor deadlines from contract terms JSONB, providing an alert feed for upcoming option deadlines and reversion triggers.

**Design**:

Alert query endpoint:
| Method | Path | Query | Response | Auth |
|--------|------|-------|----------|------|
| GET | /contracts/alerts | days_ahead=90&alert_type=option_deadline\|reversion\|expiry | list[ContractAlertResponse] | viewer+ |

```python
class ContractAlertResponse(BaseModel):
    contract_id: UUID
    contract_title: str
    contract_type: str
    alert_type: str  # option_deadline, reversion_pending, expiry_approaching
    alert_date: date
    days_remaining: int
    party_names: list[str]
    details: str
```

Alert query implementation — scans contract terms JSONB for pending deadlines:
```python
async def get_contract_alerts(db: AsyncSession, days_ahead: int = 90) -> list[ContractAlert]:
    cutoff = date.today() + timedelta(days=days_ahead)
    # Query contracts with option periods or reversion clauses
    # Use JSONB path expressions to extract dates
    query = select(Contract).where(
        Contract.status == "active",
        or_(
            Contract.expiry_date <= cutoff,
            # JSONB path: terms->'option_periods' contains pending items with deadline before cutoff
            text("EXISTS (SELECT 1 FROM jsonb_array_elements(terms->'option_periods') op WHERE op->>'status' = 'pending' AND (op->>'exercise_deadline')::date <= :cutoff)"),
            text("EXISTS (SELECT 1 FROM jsonb_array_elements(terms->'reversion_clauses') rc WHERE rc->>'status' = 'pending')"),
        )
    ).params(cutoff=cutoff)
    ...
```

**Testing**:
- `Integration: Contract with option deadline in 30 days → appears in alerts with days_ahead=90`
- `Integration: Contract with option deadline in 120 days → does NOT appear with days_ahead=90`
- `Integration: Contract with expiry in 60 days → appears as expiry_approaching`
- `Integration: Exercised option → does NOT appear in alerts`
- `Fixture: 5 contracts with varying deadlines → correct sorting by alert_date`

---

## Phase 5: Statement Ingestion Pipeline

### Purpose
Build the configurable parser pipeline that ingests royalty statements from DSPs and distributors in heterogeneous CSV/Excel formats, matches statement lines to catalog records, and flags anomalies. This is the highest-friction part of rights administration and the feature most likely to differentiate the platform. After this phase, a user can upload a Spotify quarterly statement, have it parsed using a configurable column mapping, matched against the catalog by ISRC/ISWC, and reviewed for anomalies.

### Tasks

#### 5.1 — Statement Source Configuration

**What**: CRUD for statement sources with configurable parser definitions stored as JSONB.

**Design**:

StatementSource model:
```python
class StatementSource(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "statement_sources"

    source_name: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    source_type: Mapped[str] = mapped_column(String(30), nullable=False)  # dsp, distributor, pro
    party_id: Mapped[UUID | None] = mapped_column(ForeignKey("parties.id"))
    parser_config: Mapped[dict] = mapped_column(JSONB, nullable=False)
    is_active: Mapped[bool] = mapped_column(default=True)
```

`parser_config` JSONB schema:
```json
{
    "file_format": "csv",
    "encoding": "utf-8",
    "delimiter": ",",
    "has_header": true,
    "skip_rows": 0,
    "column_mapping": {
        "isrc": {"column": "ISRC", "transform": "strip_dashes"},
        "title": {"column": "Track Title", "transform": "trim"},
        "artist": {"column": "Artist Name", "transform": "trim"},
        "territory": {"column": "Country Code", "transform": "iso_alpha2"},
        "use_type": {"column": "Sale Type", "value_map": {"Stream": "stream", "Download": "download", "Ad Supported Stream": "stream_ad"}},
        "units": {"column": "Quantity", "transform": "to_integer"},
        "unit_rate": {"column": "Per Unit Rate", "transform": "to_decimal"},
        "gross_amount": {"column": "Net Amount", "transform": "to_decimal"},
        "currency": {"column": "Currency", "transform": "uppercase"},
        "period_start": {"column": "Start Date", "format": "YYYY-MM-DD"},
        "period_end": {"column": "End Date", "format": "YYYY-MM-DD"}
    },
    "validation_rules": [
        {"field": "gross_amount", "rule": "non_negative"},
        {"field": "units", "rule": "non_negative"},
        {"field": "territory", "rule": "valid_iso_country"}
    ],
    "dedup_keys": ["isrc", "territory", "use_type", "period_start"],
    "currency_default": "USD"
}
```

Available transforms: `strip_dashes`, `trim`, `uppercase`, `lowercase`, `iso_alpha2`, `to_integer`, `to_decimal`, `to_date`.

API endpoints: standard CRUD under `/statements/sources`.

Seed data: pre-configured parser configs for Spotify, Apple Music, YouTube Music, DistroKid, TuneCore, CD Baby.

**Testing**:
- `Unit: parser_config with all required fields → passes validation`
- `Unit: parser_config with missing column_mapping → ValidationError`
- `Integration: POST statement source → 201`
- `Integration: GET /statements/sources → lists all configured sources`

---

#### 5.2 — File Upload & Batch Processing

**What**: Statement file upload, batch creation, and async processing trigger.

**Design**:

StatementBatch model:
```python
class StatementBatch(Base, UUIDMixin):
    __tablename__ = "statement_batches"

    source_id: Mapped[UUID] = mapped_column(ForeignKey("statement_sources.id"))
    file_name: Mapped[str] = mapped_column(Text, nullable=False)
    file_hash: Mapped[str | None] = mapped_column(String(64), unique=True)
    statement_period: Mapped[Range] = mapped_column(DATERANGE, nullable=False)
    original_currency: Mapped[str] = mapped_column(String(3), nullable=False)
    status: Mapped[str] = mapped_column(String(30), default="pending")
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    processed_at: Mapped[datetime | None]
    ingested_by: Mapped[UUID | None] = mapped_column(ForeignKey("users.id"))
    processing_stats: Mapped[dict] = mapped_column(JSONB, default=dict)
    anomaly_report: Mapped[dict | None] = mapped_column(JSONB)
```

Batch status lifecycle: `pending` → `parsing` → `parsed` → `matching` → `matched` → `validated` → `calculated` | `error`.

Upload endpoint:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /statements/upload | multipart: file + source_id + period_start + period_end + currency | StatementBatchResponse | accounting+ |
| GET | /statements/batches | status, source_id, period | PaginatedResponse[StatementBatchResponse] | viewer+ |
| GET | /statements/batches/{id} | — | StatementBatchDetailResponse (includes stats) | viewer+ |
| GET | /statements/batches/{id}/lines | status, matched, offset, limit | PaginatedResponse[StatementLineResponse] | viewer+ |

Upload flow:
1. Compute SHA-256 hash of uploaded file. Reject if hash already exists (duplicate detection).
2. Save file to `storage_path/{batch_id}/{filename}`.
3. Create StatementBatch row with status=`pending`.
4. Dispatch Celery task `process_statement_batch(batch_id)`.
5. Return batch response immediately (202 Accepted).

**Testing**:
- `Integration: POST /statements/upload with CSV → 202, batch created with status=pending`
- `Integration: POST /statements/upload with duplicate file → 409 (same file hash)`
- `Integration: GET /statements/batches → lists batches with processing_stats`
- `Integration: GET /statements/batches/{id}/lines → paginated line items`

---

#### 5.3 — Configurable Parser Engine

**What**: Engine that reads parser_config JSONB and transforms heterogeneous CSV/Excel files into standardized StatementLine records.

**Design**:

StatementLine model:
```python
class StatementLine(Base, UUIDMixin):
    __tablename__ = "statement_lines"

    batch_id: Mapped[UUID] = mapped_column(ForeignKey("statement_batches.id", ondelete="CASCADE"))
    line_number: Mapped[int] = mapped_column(nullable=False)

    territory_code: Mapped[str | None] = mapped_column(String(4))
    use_type: Mapped[str] = mapped_column(String(50), nullable=False)
    units: Mapped[int | None] = mapped_column(BigInteger)
    gross_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), nullable=False)
    currency_code: Mapped[str] = mapped_column(String(3), nullable=False)

    matched_recording_id: Mapped[UUID | None] = mapped_column(ForeignKey("recordings.id"))
    matched_work_id: Mapped[UUID | None] = mapped_column(ForeignKey("works.id"))
    match_confidence: Mapped[Decimal | None] = mapped_column(Numeric(5, 2))
    match_method: Mapped[str | None] = mapped_column(String(30))
    status: Mapped[str] = mapped_column(String(20), default="unmatched")

    raw_data: Mapped[dict] = mapped_column(JSONB, nullable=False)
    parsed_identifiers: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    __table_args__ = (UniqueConstraint("batch_id", "line_number"),)
```

Parser engine (`src/mrm/statements/parser.py`):
```python
class StatementParser:
    def __init__(self, config: dict):
        self.config = config
        self.column_mapping = config["column_mapping"]
        self.transforms = self._build_transforms()

    def parse_file(self, file_path: Path) -> Iterator[ParsedLine]:
        """Yields ParsedLine objects from a CSV/Excel file."""
        if self.config["file_format"] == "csv":
            df = pd.read_csv(
                file_path,
                encoding=self.config.get("encoding", "utf-8"),
                delimiter=self.config.get("delimiter", ","),
                header=0 if self.config.get("has_header", True) else None,
                skiprows=self.config.get("skip_rows", 0),
            )
        elif self.config["file_format"] == "xlsx":
            df = pd.read_excel(file_path, skiprows=self.config.get("skip_rows", 0))
        else:
            raise ValueError(f"Unsupported format: {self.config['file_format']}")

        for idx, row in df.iterrows():
            yield self._transform_row(idx + 1, row)

    def _transform_row(self, line_number: int, row: pd.Series) -> ParsedLine:
        raw_data = row.to_dict()
        parsed = {}
        for target_field, mapping in self.column_mapping.items():
            source_col = mapping["column"]
            value = row.get(source_col)
            if pd.isna(value):
                value = None
            elif "transform" in mapping:
                value = self._apply_transform(value, mapping["transform"])
            elif "value_map" in mapping:
                value = mapping["value_map"].get(str(value), str(value))
            elif "format" in mapping:
                value = self._parse_date(value, mapping["format"])
            parsed[target_field] = value
        return ParsedLine(line_number=line_number, raw_data=raw_data, **parsed)

    def _apply_transform(self, value: Any, transform: str) -> Any:
        match transform:
            case "strip_dashes": return str(value).replace("-", "")
            case "trim": return str(value).strip()
            case "uppercase": return str(value).upper()
            case "lowercase": return str(value).lower()
            case "to_integer": return int(float(value))
            case "to_decimal": return Decimal(str(value))
            case "iso_alpha2": return str(value).upper()[:2]
            case _: return value

@dataclass
class ParsedLine:
    line_number: int
    raw_data: dict
    isrc: str | None = None
    iswc: str | None = None
    title: str | None = None
    artist: str | None = None
    territory: str | None = None
    use_type: str = "stream"
    units: int | None = None
    unit_rate: Decimal | None = None
    gross_amount: Decimal = Decimal("0")
    currency: str = "USD"
    period_start: date | None = None
    period_end: date | None = None
```

**Testing**:
- `Unit: parse CSV with standard Spotify format → correct ParsedLine objects`
- `Unit: parse XLSX with DistroKid format → correct ParsedLine objects`
- `Unit: strip_dashes transform on "US-RC1-23-45678" → "USRC12345678"`
- `Unit: value_map "Ad Supported Stream" → "stream_ad"`
- `Unit: missing source column → ParsedLine with None for that field`
- `Unit: invalid numeric in gross_amount → raises ParseError with line number`
- `Fixture: tests/fixtures/statements/spotify_q1_2026.csv (100 rows, realistic data)`
- `Fixture: tests/fixtures/statements/distrokid_jan_2026.xlsx`

---

#### 5.4 — Catalog Matching Engine & Async Pipeline

**What**: Match parsed statement lines to catalog works/recordings by ISRC, ISWC, or fuzzy title+artist. Run as Celery task.

**Design**:

Matcher (`src/mrm/statements/matcher.py`):
```python
class CatalogMatcher:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def match_line(self, line: ParsedLine) -> MatchResult:
        # Priority 1: ISRC exact match
        if line.isrc:
            result = await self._match_by_isrc(line.isrc)
            if result:
                return MatchResult(recording_id=result.recording_id, work_id=result.work_id, method="isrc", confidence=Decimal("100"))

        # Priority 2: ISWC exact match
        if line.iswc:
            result = await self._match_by_iswc(line.iswc)
            if result:
                return MatchResult(work_id=result.work_id, method="iswc", confidence=Decimal("100"))

        # Priority 3: Fuzzy title + artist match via pg_trgm
        if line.title:
            result = await self._match_by_title_artist(line.title, line.artist)
            if result and result.confidence >= 80:
                return result

        return MatchResult(method="none", confidence=Decimal("0"))

    async def _match_by_isrc(self, isrc: str) -> MatchResult | None:
        recording = await self.db.execute(
            select(Recording).where(Recording.isrc == isrc)
        )
        rec = recording.scalar_one_or_none()
        if not rec:
            return None
        # Find linked work
        link = await self.db.execute(
            select(WorkRecording.work_id).where(WorkRecording.recording_id == rec.id, WorkRecording.is_primary == True)
        )
        work_id = link.scalar_one_or_none()
        return MatchResult(recording_id=rec.id, work_id=work_id, method="isrc", confidence=Decimal("100"))

    async def _match_by_title_artist(self, title: str, artist: str | None) -> MatchResult | None:
        query = text("""
            SELECT r.id AS recording_id, wr.work_id,
                   similarity(r.title, :title) AS title_sim,
                   CASE WHEN p.legal_name IS NOT NULL
                        THEN similarity(p.legal_name, :artist) ELSE 0 END AS artist_sim
            FROM recordings r
            LEFT JOIN parties p ON p.id = r.primary_artist_id
            LEFT JOIN work_recordings wr ON wr.recording_id = r.id AND wr.is_primary = TRUE
            WHERE similarity(r.title, :title) > 0.3
            ORDER BY similarity(r.title, :title) DESC
            LIMIT 1
        """)
        result = await self.db.execute(query, {"title": title, "artist": artist or ""})
        row = result.first()
        if not row:
            return None
        confidence = Decimal(str((row.title_sim * 70 + row.artist_sim * 30)))
        return MatchResult(recording_id=row.recording_id, work_id=row.work_id, method="title_artist", confidence=min(confidence, Decimal("99.99")))
```

Celery pipeline (`src/mrm/statements/tasks.py`):
```python
@celery_app.task(bind=True, max_retries=3)
def process_statement_batch(self, batch_id: str) -> None:
    """Parse → Match → Validate a statement batch."""
    # 1. Parse
    update_batch_status(batch_id, "parsing")
    source = get_statement_source(batch_id)
    parser = StatementParser(source.parser_config)
    lines = list(parser.parse_file(get_batch_file_path(batch_id)))
    bulk_insert_statement_lines(batch_id, lines)
    update_batch_status(batch_id, "parsed")

    # 2. Match
    update_batch_status(batch_id, "matching")
    matcher = CatalogMatcher(get_sync_db())
    for line in get_unmatched_lines(batch_id):
        result = matcher.match_line_sync(line)
        update_line_match(line.id, result)
    update_batch_status(batch_id, "matched")

    # 3. Validate
    update_batch_status(batch_id, "validating")
    stats = compute_batch_stats(batch_id)
    update_batch_processing_stats(batch_id, stats)
    update_batch_status(batch_id, "validated")
```

Batch stats computed after matching:
```json
{
    "total_lines": 45230,
    "matched_lines": 44100,
    "unmatched_lines": 1130,
    "total_amount": 125430.75,
    "match_rate": 97.5,
    "parse_errors": 0,
    "anomaly_count": 0,
    "duration_ms": 15400
}
```

**Testing**:
- `Unit: match_by_isrc with existing ISRC → MatchResult with confidence=100, method="isrc"`
- `Unit: match_by_isrc with unknown ISRC → None`
- `Unit: match_by_title_artist "Midnight Rain" + "Artist A" → match with confidence > 80`
- `Unit: match_by_title_artist with no similar titles → None`
- `Integration: upload fixture CSV → Celery task parses all lines, match rate > 0%`
- `Integration: re-upload same file → 409 (duplicate hash)`
- `Integration: GET /statements/batches/{id} after processing → status=validated, stats populated`
- `Integration: GET /statements/batches/{id}/lines?status=unmatched → returns only unmatched lines`
- `Fixture: Spotify CSV with known ISRCs matching test catalog → 100% match rate`
- `E2E: Upload → parse → match → validate → batch moves through all status stages`

---

## Phase 6: Royalty Calculation Engine

### Purpose
Build the core calculation engine that takes matched statement lines, resolves the correct ownership shares per territory and use type, applies currency conversion, admin fees, withholding tax, and advance recoupment, and produces payable amounts per rights holder. After this phase, the system can transform a DSP statement into a complete royalty breakdown showing exactly how much each party is owed and why.

### Tasks

#### 6.1 — FX Rate Ingestion

**What**: Daily foreign exchange rate fetching from ECB (free, no API key) or OpenExchangeRates, with historical rate preservation.

**Design**:

FXRate model:
```python
class FXRate(Base, UUIDMixin):
    __tablename__ = "fx_rates"

    rate_date: Mapped[date] = mapped_column(nullable=False)
    from_currency: Mapped[str] = mapped_column(String(3), nullable=False)
    to_currency: Mapped[str] = mapped_column(String(3), nullable=False)
    rate: Mapped[Decimal] = mapped_column(Numeric(16, 8), nullable=False)
    source: Mapped[str] = mapped_column(String(50), nullable=False)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    __table_args__ = (UniqueConstraint("rate_date", "from_currency", "to_currency"),)
```

FX fetcher (`src/mrm/royalties/fx.py`):
```python
class ECBFXFetcher:
    ECB_URL = "https://data-api.ecb.europa.eu/service/data/EXR/D..EUR.SP00.A"

    async def fetch_rates(self, target_date: date) -> list[FXRate]:
        """Fetches ECB daily rates. EUR is base currency; derives cross rates for USD base."""
        async with httpx.AsyncClient() as client:
            resp = await client.get(self.ECB_URL, params={"startPeriod": str(target_date), "endPeriod": str(target_date)})
            resp.raise_for_status()
            # Parse XML response → list of (currency, rate_vs_eur)
            # Convert to USD base: usd_rate = rate_vs_eur / eur_to_usd
            ...

async def get_fx_rate(db: AsyncSession, rate_date: date, from_currency: str, to_currency: str) -> Decimal:
    """Returns the FX rate, fetching from external source if not in DB."""
    if from_currency == to_currency:
        return Decimal("1")
    rate = await db.execute(
        select(FXRate.rate).where(
            FXRate.rate_date == rate_date,
            FXRate.from_currency == from_currency,
            FXRate.to_currency == to_currency
        )
    )
    if result := rate.scalar_one_or_none():
        return result
    # Fetch and cache
    await fetch_and_store_rates(db, rate_date)
    # Retry lookup
    ...
```

Celery task for daily FX rate prefetch:
```python
@celery_app.task
def prefetch_fx_rates() -> None:
    """Fetch today's FX rates from ECB. Scheduled daily at 16:00 UTC (after ECB publishes)."""
    ...
```

API endpoints:
| Method | Path | Query | Response | Auth |
|--------|------|-------|----------|------|
| GET | /royalties/fx-rates | date, from_currency, to_currency | FXRateResponse | viewer+ |
| POST | /royalties/fx-rates | FXRateCreate (manual entry) | FXRateResponse | accounting+ |

**Testing**:
- `Unit: get_fx_rate("USD", "USD") → Decimal("1")`
- `Unit: get_fx_rate with cached rate → returns from DB without HTTP call`
- `Integration (mocked HTTP): ECB fetch → parses XML, stores rates, returns correct USD→GBP rate`
- `Integration: POST manual FX rate → stored and retrievable`
- `Integration: duplicate rate insert → updates existing (upsert behavior)`

---

#### 6.2 — Core Royalty Calculation Engine

**What**: Engine that processes matched statement lines, resolves shares with territory overrides, converts currency, and produces per-party royalty calculations.

**Design**:

RoyaltyCalculation model:
```python
class RoyaltyCalculation(Base, UUIDMixin):
    __tablename__ = "royalty_calculations"

    statement_line_id: Mapped[UUID] = mapped_column(ForeignKey("statement_lines.id"))
    work_share_id: Mapped[UUID | None] = mapped_column(ForeignKey("work_shares.id"))
    recording_share_id: Mapped[UUID | None] = mapped_column(ForeignKey("recording_shares.id"))
    party_id: Mapped[UUID] = mapped_column(ForeignKey("parties.id"))

    gross_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), nullable=False)
    original_currency: Mapped[str] = mapped_column(String(3), nullable=False)
    fx_rate_id: Mapped[UUID | None] = mapped_column(ForeignKey("fx_rates.id"))
    converted_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), nullable=False)
    settlement_currency: Mapped[str] = mapped_column(String(3), nullable=False)
    share_percentage: Mapped[Decimal] = mapped_column(Numeric(7, 4), nullable=False)
    rights_type: Mapped[str] = mapped_column(String(30), nullable=False)
    royalty_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), nullable=False)
    admin_fee_pct: Mapped[Decimal] = mapped_column(Numeric(5, 2), default=0)
    admin_fee_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), default=0)
    withholding_tax_pct: Mapped[Decimal] = mapped_column(Numeric(5, 2), default=0)
    withholding_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), default=0)
    net_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), nullable=False)
    recoupable: Mapped[bool] = mapped_column(default=False)
    recouped_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), default=0)
    payable_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), nullable=False)
    contract_id: Mapped[UUID | None] = mapped_column(ForeignKey("contracts.id"))
    calculation_period: Mapped[Range | None] = mapped_column(DATERANGE)
    calculated_at: Mapped[datetime] = mapped_column(server_default=func.now())
    calculation_inputs: Mapped[dict] = mapped_column(JSONB, default=dict)
```

Calculation engine (`src/mrm/royalties/engine.py`):
```python
class RoyaltyEngine:
    def __init__(self, db: AsyncSession, settlement_currency: str = "USD"):
        self.db = db
        self.settlement_currency = settlement_currency

    async def calculate_batch(self, batch_id: UUID, fx_rate_date: date) -> CalculationSummary:
        """Calculate royalties for all matched lines in a statement batch."""
        lines = await self._get_matched_lines(batch_id)
        calculations = []
        for line in lines:
            calcs = await self._calculate_line(line, fx_rate_date)
            calculations.extend(calcs)
        await self._bulk_insert_calculations(calculations)
        return self._summarize(calculations)

    async def _calculate_line(self, line: StatementLine, fx_rate_date: date) -> list[RoyaltyCalculation]:
        """For a single statement line, calculate each rights holder's share."""
        calcs = []

        # Get FX rate
        fx_rate = await get_fx_rate(self.db, fx_rate_date, line.currency_code, self.settlement_currency)
        converted = line.gross_amount * fx_rate

        # Determine rights type from use_type
        rights_type = self._map_use_type_to_rights(line.use_type)

        # Get active shares for the matched work/recording at usage date
        if line.matched_work_id and rights_type in ("performance", "mechanical", "sync", "print"):
            shares = await self._get_active_work_shares(line.matched_work_id, line.territory_code)
            for share in shares:
                effective_pct = get_effective_share(share, line.territory_code or "WW", rights_type)
                calcs.append(self._build_calculation(line, share, effective_pct, rights_type, converted, fx_rate))

        if line.matched_recording_id and rights_type in ("master", "neighboring"):
            rec_shares = await self._get_active_recording_shares(line.matched_recording_id, line.territory_code)
            for share in rec_shares:
                share_key = "master_share" if rights_type == "master" else "neighboring_share"
                effective_pct = getattr(share, share_key)
                calcs.append(self._build_calculation(line, share, effective_pct, rights_type, converted, fx_rate))

        return calcs

    def _build_calculation(self, line, share, effective_pct, rights_type, converted, fx_rate) -> dict:
        royalty = converted * (effective_pct / Decimal("100"))
        # TODO Phase 6.3: apply admin fees, withholding, recoupment from contract terms
        return {
            "statement_line_id": line.id,
            "work_share_id": share.id if hasattr(share, "work_id") else None,
            "recording_share_id": share.id if hasattr(share, "recording_id") else None,
            "party_id": share.party_id,
            "gross_amount": line.gross_amount,
            "original_currency": line.currency_code,
            "converted_amount": converted,
            "settlement_currency": self.settlement_currency,
            "share_percentage": effective_pct,
            "rights_type": rights_type,
            "royalty_amount": royalty,
            "net_amount": royalty,
            "payable_amount": royalty,
            "calculated_at": datetime.utcnow(),
            "calculation_inputs": {
                "share_version": share.version,
                "territory": line.territory_code,
                "fx_rate": str(fx_rate),
                "fx_rate_date": str(fx_rate.rate_date) if hasattr(fx_rate, "rate_date") else None,
            }
        }

    def _map_use_type_to_rights(self, use_type: str) -> str:
        MAPPING = {
            "stream": "performance", "download": "mechanical", "ringtone": "mechanical",
            "broadcast_radio": "performance", "broadcast_tv": "performance",
            "public_performance": "performance", "sync_fee": "sync",
            "mechanical": "mechanical", "neighboring": "neighboring",
            "user_generated_content": "performance", "social_media": "performance",
        }
        return MAPPING.get(use_type, "performance")
```

API endpoints:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /royalties/calculate | {batch_ids: list[UUID], fx_rate_date: date} | CalculationSummaryResponse (202, async) | accounting+ |
| GET | /royalties/calculations | party_id, work_id, period, rights_type | PaginatedResponse[RoyaltyCalculationResponse] | viewer+ |
| GET | /royalties/summary | party_id, period_year, period_quarter, territory | RoyaltySummaryResponse | viewer+ |

Celery task:
```python
@celery_app.task
def run_royalty_calculation(batch_ids: list[str], fx_rate_date: str, settlement_currency: str = "USD") -> dict:
    ...
```

**Testing**:
- `Unit: _map_use_type_to_rights("stream") → "performance"`
- `Unit: _map_use_type_to_rights("download") → "mechanical"`
- `Unit: _build_calculation with 25% share on $100 → royalty_amount = $25.00`
- `Unit: _build_calculation with FX rate 0.85 → converted_amount correct`
- `Integration: calculate_batch with 3 parties sharing a work → 3 calculations per line`
- `Integration: territory override in DE → different share percentage used for DE lines`
- `Integration: excluded territory → share = 0, no calculation generated`
- `Integration: GET /royalties/summary?party_id=X → aggregated totals per rights type`
- `E2E: Upload statement → parse → match → calculate → correct royalty amounts per party`

---

#### 6.3 — Admin Fees, Withholding Tax & Advance Recoupment

**What**: Apply contract-based deductions to raw royalty amounts: admin fees, withholding tax, and advance recoupment.

**Design**:

Extend `_build_calculation` to look up contract terms:
```python
async def _apply_deductions(self, calc: dict, share, line) -> dict:
    # Find governing contract for this party + work
    contract = await self._find_governing_contract(share.party_id, line.matched_work_id)
    if not contract:
        return calc

    calc["contract_id"] = contract.id
    terms = contract.terms

    # Admin fee
    admin_fee_pct = Decimal(str(terms.get("admin_fee", terms.get("royalty_rates", {}).get("admin_fee", 0))))
    if admin_fee_pct > 0:
        calc["admin_fee_pct"] = admin_fee_pct
        calc["admin_fee_amount"] = calc["royalty_amount"] * admin_fee_pct / Decimal("100")
        calc["net_amount"] = calc["royalty_amount"] - calc["admin_fee_amount"]

    # Withholding tax (based on party country)
    wht_pct = await self._get_withholding_rate(share.party_id, line.territory_code)
    if wht_pct > 0:
        calc["withholding_tax_pct"] = wht_pct
        calc["withholding_amount"] = calc["net_amount"] * wht_pct / Decimal("100")
        calc["net_amount"] -= calc["withholding_amount"]

    # Advance recoupment
    advance = terms.get("advance", {})
    if advance.get("recoupable") and not advance.get("fully_recouped", False):
        remaining = Decimal(str(advance["amount"])) - Decimal(str(advance.get("recouped_to_date", 0)))
        if remaining > 0:
            recoup = min(calc["net_amount"], remaining)
            calc["recoupable"] = True
            calc["recouped_amount"] = recoup
            calc["payable_amount"] = calc["net_amount"] - recoup
            # Update contract advance tracking
            await self._update_advance_recoupment(contract.id, recoup)
        else:
            calc["payable_amount"] = calc["net_amount"]
    else:
        calc["payable_amount"] = calc["net_amount"]

    calc["calculation_inputs"]["admin_fee_pct"] = str(admin_fee_pct)
    calc["calculation_inputs"]["withholding_pct"] = str(wht_pct)
    calc["calculation_inputs"]["advance_remaining"] = str(remaining) if advance.get("recoupable") else "0"
    return calc
```

**Testing**:
- `Unit: 15% admin fee on $100 royalty → admin_fee_amount=$15, net=$85`
- `Unit: 30% withholding on $85 net → withholding=$25.50, net=$59.50`
- `Unit: $500 advance remaining, $100 payable → recouped=$100, payable=$0`
- `Unit: $50 advance remaining, $100 payable → recouped=$50, payable=$50`
- `Unit: No contract found → no deductions applied, payable = royalty_amount`
- `Integration: full calculation with all deductions → correct cascading math`
- `Integration: advance fully recouped → subsequent calculations have recouped_amount=0`

---

## Phase 7: PRO Registration (CWR)

### Purpose
Implement CWR 2.2 file generation for registering musical works with performing rights organisations (ASCAP, BMI, SESAC, PRS, SOCAN, GEMA). After this phase, a publisher can select works from the catalog, generate a properly formatted CWR file, track submission status per society, and parse acknowledgement files.

### Tasks

#### 7.1 — CWR 2.2 File Generation

**What**: Generate CWR-compliant registration files from work and share data following CISAC CWR 2.2 specification.

**Design**:

CWR generator (`src/mrm/pro/cwr.py`):
```python
class CWRGenerator:
    CWR_VERSION = "02.20"

    def generate(self, works: list[WorkWithShares], submitter: Party, receiver: Party) -> str:
        """Generate a CWR file for a batch of works."""
        lines = []
        lines.append(self._hdr_record(submitter, receiver))
        lines.append(self._grh_record())

        for seq, work in enumerate(works, 1):
            lines.extend(self._work_transaction(work, seq))

        lines.append(self._grt_record(len(works)))
        lines.append(self._trl_record(len(works)))
        return "\r\n".join(lines)

    def _hdr_record(self, submitter: Party, receiver: Party) -> str:
        """HDR - Header record (80 chars fixed width)."""
        return (
            "HDRPB"                              # Record type + sender type
            + f"{submitter.ipi_name_number:>11}"  # Sender IPI
            + f"{submitter.legal_name:<45}"       # Sender name
            + " " * 5                             # EDI standard version (blank)
            + datetime.now().strftime("%Y%m%d")   # Creation date
            + datetime.now().strftime("%H%M%S")   # Creation time
            + datetime.now().strftime("%Y%m%d")   # Transmission date
            + f"{'':>5}"                          # Character set
            + f"{'':>10}"                         # Sender sequence
            + f"{receiver.legal_name:<45}"        # Receiver name
        ).ljust(80)

    def _work_transaction(self, work: WorkWithShares, seq: int) -> list[str]:
        """Generate NWR + SPU/SPT + SWR/SWT + PER + REC records for one work."""
        lines = []
        # NWR - New Work Registration
        lines.append(self._nwr_record(work, seq))
        # SPU/SPT - Publisher shares with territories
        for share in work.publisher_shares:
            lines.append(self._spu_record(share, seq))
            for territory in self._expand_territories(share):
                lines.append(self._spt_record(share, territory, seq))
        # SWR/SWT - Writer shares with territories
        for share in work.writer_shares:
            lines.append(self._swr_record(share, seq))
            for territory in self._expand_territories(share):
                lines.append(self._swt_record(share, territory, seq))
        return lines

    def _nwr_record(self, work: WorkWithShares, seq: int) -> str:
        """NWR - New Work Registration record."""
        # CWR 2.2 NWR format: fixed-width fields
        return (
            "NWR"
            + f"{seq:>8}"                         # Transaction sequence
            + f"{0:>8}"                           # Record sequence
            + f"{work.title:<60}"                 # Work title
            + f"{'EN':<2}"                        # Language code
            + f"{'':>11}"                         # Submitter work ID
            + f"{work.iswc or '':>11}"            # ISWC
            + f"{'':>14}"                         # Copyright date
            + f"{'ORI':<3}"                       # Version type
            + f"{'MTX':<3}"                       # Music/text relationship
            + f"{'ORI':<3}"                       # Music arrangement
            + f"{'ORI':<3}"                       # Lyric adaptation
            # ... additional fields per CWR 2.2 spec
        ).ljust(399)
```

**Testing**:
- `Unit: generate() with 1 work, 2 shares → valid CWR file with HDR + GRH + NWR + SPU + SPT + SWR + SWT + GRT + TRL`
- `Unit: HDR record is exactly 80 characters`
- `Unit: NWR record contains correct ISWC in correct position`
- `Unit: SPU record contains correct IPI and share percentages`
- `Unit: Territory codes in SPT records use CISAC codes, not ISO`
- `Fixture: tests/fixtures/cwr/sample_nwr.cwr — reference CWR file for comparison`
- `Unit: generate() with work missing ISWC → ISWC field blank-filled`

---

#### 7.2 — Registration Record Management & Status Tracking

**What**: Track CWR submissions to each PRO with status lifecycle.

**Design**:

ProRegistration model:
```python
class ProRegistration(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "pro_registrations"

    work_id: Mapped[UUID] = mapped_column(ForeignKey("works.id"))
    pro_party_id: Mapped[UUID] = mapped_column(ForeignKey("parties.id"))
    registration_type: Mapped[str] = mapped_column(String(10), nullable=False)  # NWR, REV
    cwr_version: Mapped[str] = mapped_column(String(5), default="2.2")
    status: Mapped[str] = mapped_column(String(30), default="draft")
    submission_date: Mapped[date | None]
    acknowledgement_date: Mapped[date | None]
    society_work_id: Mapped[str | None] = mapped_column(String(30))
    cwr_transaction: Mapped[dict | None] = mapped_column(JSONB)
    acknowledgement: Mapped[dict | None] = mapped_column(JSONB)
```

Status lifecycle: `draft` → `submitted` → `acknowledged` → `accepted` | `rejected` | `conflict`.

API endpoints:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /pro/registrations/generate | {work_ids: list[UUID], pro_party_id: UUID} | {cwr_file_url: str, registrations: list[RegistrationResponse]} | publisher+ |
| GET | /pro/registrations | work_id, pro_party_id, status | PaginatedResponse[RegistrationResponse] | viewer+ |
| PATCH | /pro/registrations/{id} | {status, society_work_id, acknowledgement} | RegistrationResponse | publisher+ |
| GET | /pro/registrations/dashboard | — | RegistrationDashboardResponse | viewer+ |

Dashboard response:
```python
class RegistrationDashboardResponse(BaseModel):
    total_works: int
    registered_works: int
    pending_works: int
    by_pro: list[ProRegistrationSummary]  # [{pro_name, submitted, accepted, rejected, pending}]
    unregistered_works: list[UnregisteredWorkSummary]
```

**Testing**:
- `Integration: POST generate → CWR file created, download URL returned, registration records created`
- `Integration: PATCH with accepted status → society_work_id stored`
- `Integration: GET dashboard → correct counts per PRO`
- `Integration: works not registered with any PRO → appear in unregistered_works list`

---

#### 7.3 — Acknowledgement File Parsing

**What**: Parse CWR acknowledgement files returned by PROs and update registration status.

**Design**:

```python
class CWRAcknowledgementParser:
    def parse(self, content: str) -> list[AcknowledgementResult]:
        """Parse a CWR acknowledgement file and return results per transaction."""
        results = []
        for line in content.split("\r\n"):
            if line.startswith("ACK"):
                result = self._parse_ack_record(line)
                results.append(result)
        return results

    def _parse_ack_record(self, line: str) -> AcknowledgementResult:
        status_code = line[5:7]  # AS=accepted, RJ=rejected, NP=no participation, etc.
        society_work_id = line[7:21].strip()
        transaction_id = line[21:29].strip()
        return AcknowledgementResult(
            transaction_id=transaction_id,
            status="accepted" if status_code == "AS" else "rejected" if status_code == "RJ" else "conflict",
            society_work_id=society_work_id if status_code == "AS" else None,
            raw_status_code=status_code,
        )
```

Upload endpoint:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /pro/acknowledgements/upload | multipart: file + pro_party_id | list[AcknowledgementResultResponse] | publisher+ |

**Testing**:
- `Unit: parse ACK with AS status → accepted, society_work_id extracted`
- `Unit: parse ACK with RJ status → rejected, rejection reason captured`
- `Integration: upload acknowledgement file → matching registrations updated with status + society_work_id`
- `Fixture: tests/fixtures/cwr/sample_ack.cwr — realistic acknowledgement file`

---

## Phase 8: Frontend Application

### Purpose
Build the React web UI that exposes all backend functionality through an intuitive dashboard. After this phase, all operations — catalog management, split editing, statement upload, royalty review, PRO registration — are accessible through the browser without requiring API calls.

### Tasks

#### 8.1 — App Shell, Routing & Authentication UI

**What**: Vite + React project setup, app shell with navigation, login/register pages, and authenticated routing.

**Design**:

Routes:
```
/login                      → LoginPage
/register                   → RegisterPage
/dashboard                  → DashboardPage (overview metrics)
/catalog/works              → WorkListPage
/catalog/works/:id          → WorkDetailPage
/catalog/recordings         → RecordingListPage
/catalog/recordings/:id     → RecordingDetailPage
/catalog/parties            → PartyListPage
/catalog/parties/:id        → PartyDetailPage
/rights/works/:id/shares    → WorkSharesPage
/contracts                  → ContractListPage
/contracts/:id              → ContractDetailPage
/statements                 → StatementBatchListPage
/statements/:id             → StatementBatchDetailPage
/royalties                  → RoyaltySummaryPage
/royalties/calculations     → CalculationListPage
/pro/registrations          → ProRegistrationDashboard
/pro/registrations/:id      → ProRegistrationDetailPage
/settings                   → SettingsPage
```

Layout: sidebar navigation with role-based menu items. Accounting-only items hidden from viewer role.

Auth: store JWT in httpOnly cookie or localStorage; TanStack Query with auth header interceptor; redirect to /login on 401.

API client generated from OpenAPI spec using `openapi-typescript-codegen` or `orval`.

**Testing**:
- `E2E (Playwright): navigate to /login → enter credentials → redirected to /dashboard`
- `E2E: unauthenticated access to /catalog/works → redirected to /login`
- `E2E: viewer role → cannot see "Upload Statement" menu item`
- `Component: Sidebar renders correct items for each role`

---

#### 8.2 — Catalog Management UI

**What**: Work, recording, party, and release list/detail pages with search, create, and edit forms.

**Design**:

WorkListPage:
- Data table with columns: Title, ISWC, Genre, Status, Recordings, Created
- Search bar (triggers `/catalog/search?q=...`)
- Filters: status, genre, AI contribution
- Pagination via TanStack Query infinite scroll or page numbers
- "Create Work" button → modal form

WorkDetailPage:
- Work metadata display + edit form
- Linked recordings list with "Link Recording" button
- Active shares table (from `/rights/works/{id}/shares`)
- Share totals warning banner if != 100%
- PRO registration status per society

PartyListPage:
- Data table with columns: Name, Type, IPI, Country, Status
- Search + type filter
- "Create Party" button → modal form

PartyDetailPage:
- Party info + identifiers (JSONB rendered as editable key-value pairs)
- Works owned (shares list)
- Recordings owned
- Contracts list
- Payment info (bank accounts, masked)

**Testing**:
- `E2E: Create work via UI → appears in work list`
- `E2E: Search "midnight" → correct results shown`
- `E2E: Link recording to work → recording appears in work detail`
- `Component: WorkForm validates ISWC format client-side`
- `Component: PartyForm renders type-appropriate identifier fields`

---

#### 8.3 — Split Management & Contract Management UI

**What**: Split sheet editor with version history, territory override configuration, and contract management forms.

**Design**:

WorkSharesPage:
- Table of active shares: Party, Role, Performance%, Mechanical%, Sync%, Print%, Controlled, Effective From
- "Add Share" button → form with party selector, role, percentages
- "Revise" button per row → form pre-filled with current values, requires reason
- "History" button per row → modal showing all versions
- Share totals bar chart (visual indicator of 100% target)
- Territory overrides: expandable section per share showing territory-specific percentages

ContractListPage:
- Data table: Title, Type, Status, Effective Date, Expiry, Parties
- Filters: type, status, expiring_before
- "Create Contract" button → multi-step form (type selection → parties → terms → territories)

ContractDetailPage:
- Contract metadata
- Parties involved
- Covered works
- Terms rendered based on contract_type (co-pub shows advance/rates/options; sync shows licence fee/usage)
- Option period timeline (visual)
- Reversion triggers with status indicators
- Document attachments list with upload

Alert banner on dashboard: upcoming option deadlines and contract expirations.

**Testing**:
- `E2E: Add share → share appears in table, totals update`
- `E2E: Revise share → new version created, old version in history`
- `E2E: Add territory override for DE → visible in territory section`
- `E2E: Create co-publishing contract → terms form shows advance, rates, options`
- `Component: Share totals bar shows red when > 100%`
- `Component: Option period timeline renders correctly`

---

#### 8.4 — Statement Upload, Royalty Dashboard & PRO Registration UI

**What**: Statement upload flow, royalty results dashboard, and PRO registration interface.

**Design**:

StatementBatchListPage:
- Table: Source, File, Period, Status, Match Rate, Total Amount, Uploaded
- Status badges with progress indicator for in-progress batches (poll via TanStack Query refetchInterval)
- "Upload Statement" button → form: select source, upload file, set period/currency

StatementBatchDetailPage:
- Batch summary: total lines, matched/unmatched counts, total amount
- Anomaly report (if present)
- Line items table with filters: status (matched/unmatched), territory, use_type
- Per-line detail: raw data, matched work/recording, confidence score
- "Recalculate" button to re-run matching

RoyaltySummaryPage:
- Revenue dashboard with filters: period, territory, rights_type, party
- Charts: revenue by territory (map or bar), revenue by rights type (pie), revenue by period (line)
- Summary table: Party, Total Gross, Total Royalties, Admin Fees, Withholding, Net, Payable
- Drill-down: click party → per-work breakdown
- "Run Calculation" button → select batches, FX date → triggers calculation

ProRegistrationDashboard:
- Summary cards: Total Works, Registered, Pending, Rejected
- Per-PRO status table
- Unregistered works list with multi-select + "Generate CWR" button
- Upload acknowledgement file button
- Timeline of recent submissions/acknowledgements

**Testing**:
- `E2E: Upload CSV → batch appears in list → status progresses to validated`
- `E2E: Run royalty calculation → results appear in dashboard`
- `E2E: Royalty summary filters by territory → chart updates`
- `E2E: Generate CWR for selected works → download link provided`
- `E2E: Upload acknowledgement → registration statuses updated`
- `Component: Revenue by territory chart renders with real data`
- `Component: Statement batch status badge updates via polling`

---

## Phase 9: Audit Trail & Reporting

### Purpose
Implement the immutable audit log that captures every ownership change, payment run, and statement adjustment, plus reporting capabilities for revenue analytics and data export. After this phase, the system satisfies the audit requirements critical for rights administration compliance.

### Tasks

#### 9.1 — Audit Log Implementation

**What**: Append-only audit log capturing all state changes across the system.

**Design**:

AuditLog model:
```python
class AuditLog(Base, UUIDMixin):
    __tablename__ = "audit_log"

    event_type: Mapped[str] = mapped_column(String(50), nullable=False)
    entity_type: Mapped[str] = mapped_column(String(50), nullable=False)
    entity_id: Mapped[UUID] = mapped_column(nullable=False)
    actor_id: Mapped[UUID | None] = mapped_column(ForeignKey("users.id"))
    actor_ip: Mapped[str | None] = mapped_column(String(45))
    changes: Mapped[dict] = mapped_column(JSONB, nullable=False)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

Event types: `share_created`, `share_revised`, `share_superseded`, `statement_ingested`, `statement_matched`, `calculation_run`, `payment_approved`, `payment_sent`, `contract_created`, `contract_modified`, `registration_submitted`, `registration_acknowledged`, `user_login`, `permission_change`.

Audit service — called from all mutation service functions:
```python
async def log_audit(
    db: AsyncSession,
    event_type: str,
    entity_type: str,
    entity_id: UUID,
    actor_id: UUID | None,
    old_values: dict | None = None,
    new_values: dict | None = None,
    metadata: dict | None = None,
) -> None:
    entry = AuditLog(
        event_type=event_type,
        entity_type=entity_type,
        entity_id=entity_id,
        actor_id=actor_id,
        changes={"old": old_values, "new": new_values, **(metadata or {})},
    )
    db.add(entry)
```

Retroactive integration: add `log_audit` calls to all existing service functions in catalog, rights, contracts, statements, royalties, and pro modules.

API endpoints:
| Method | Path | Query | Response | Auth |
|--------|------|-------|----------|------|
| GET | /audit | entity_type, entity_id, event_type, actor_id, from_date, to_date | PaginatedResponse[AuditLogResponse] | manager+ |
| GET | /audit/entity/{type}/{id} | — | list[AuditLogResponse] (full history for one entity) | manager+ |

**Testing**:
- `Integration: create work → audit log entry with event_type=work_created`
- `Integration: revise share → audit log captures old and new share values`
- `Integration: GET /audit?entity_type=work_shares&entity_id=X → returns share history`
- `Integration: audit_log table rejects UPDATE/DELETE (enforced via RLS or trigger)`

---

#### 9.2 — Revenue Analytics & Reporting Views

**What**: Materialized views and API endpoints for revenue dashboards and analytics.

**Design**:

Materialized view:
```sql
CREATE MATERIALIZED VIEW mv_royalty_summary AS
SELECT
    rc.party_id,
    p.legal_name AS party_name,
    sl.territory_code,
    sl.use_type,
    rc.rights_type,
    rc.settlement_currency,
    date_trunc('month', lower(rc.calculation_period)) AS period_month,
    EXTRACT(YEAR FROM lower(rc.calculation_period))::int AS period_year,
    EXTRACT(QUARTER FROM lower(rc.calculation_period))::int AS period_quarter,
    SUM(rc.gross_amount) AS total_gross,
    SUM(rc.royalty_amount) AS total_royalty,
    SUM(rc.admin_fee_amount) AS total_admin_fees,
    SUM(rc.withholding_amount) AS total_withholding,
    SUM(rc.net_amount) AS total_net,
    SUM(rc.payable_amount) AS total_payable,
    SUM(sl.units) AS total_units,
    COUNT(*) AS line_count
FROM royalty_calculations rc
JOIN statement_lines sl ON sl.id = rc.statement_line_id
JOIN parties p ON p.id = rc.party_id
GROUP BY rc.party_id, p.legal_name, sl.territory_code, sl.use_type,
         rc.rights_type, rc.settlement_currency,
         date_trunc('month', lower(rc.calculation_period)),
         EXTRACT(YEAR FROM lower(rc.calculation_period)),
         EXTRACT(QUARTER FROM lower(rc.calculation_period));

CREATE INDEX idx_mvrs_party ON mv_royalty_summary (party_id);
CREATE INDEX idx_mvrs_period ON mv_royalty_summary (period_year, period_quarter);
```

Refresh after each royalty calculation run (triggered by Celery task completion).

Analytics API endpoints:
| Method | Path | Query | Response | Auth |
|--------|------|-------|----------|------|
| GET | /reports/revenue/by-territory | period_year, period_quarter, party_id | list[{territory, total_gross, total_payable}] | viewer+ |
| GET | /reports/revenue/by-rights-type | period_year, period_quarter, party_id | list[{rights_type, total_gross, total_payable}] | viewer+ |
| GET | /reports/revenue/by-period | party_id, from_year, to_year | list[{period, total_gross, total_payable}] | viewer+ |
| GET | /reports/revenue/top-works | period_year, period_quarter, limit | list[{work_title, total_gross, total_payable}] | viewer+ |

**Testing**:
- `Integration: after royalty calculation → mv_royalty_summary has correct aggregations`
- `Integration: GET /reports/revenue/by-territory → returns territory breakdown`
- `Integration: revenue report for party with no calculations → empty result, not error`

---

#### 9.3 — Data Export (CSV, PDF Statements)

**What**: Export royalty statements and reports as CSV or PDF for distribution to rights holders.

**Design**:

Export endpoints:
| Method | Path | Query | Response | Auth |
|--------|------|-------|----------|------|
| GET | /reports/export/csv | party_id, period_year, period_quarter | CSV file download | accounting+ |
| GET | /reports/export/statement | party_id, period_year, period_quarter | PDF file download | accounting+ |

CSV export: standard columnar format with headers: Work Title, ISWC, Territory, Use Type, Rights Type, Gross, Share%, Royalty, Admin Fee, Withholding, Net, Payable.

PDF statement: formatted royalty statement with party details, period, per-work breakdown, and totals. Generated using `reportlab` or `weasyprint`.

**Testing**:
- `Integration: GET CSV export → valid CSV with correct rows and totals`
- `Integration: GET PDF export → valid PDF file (check Content-Type and file size > 0)`
- `Integration: export for party with no data → empty CSV with headers only`

---

## Phase 10: Payment Disbursement

### Purpose
Integrate with payment processors (Stripe Connect) to automate royalty disbursements to rights holders. After this phase, the full cycle is complete: statement → calculation → payment.

### Tasks

#### 10.1 — Stripe Connect Integration

**What**: Connect rights holders' bank accounts via Stripe Connect and enable programmatic transfers.

**Design**:

Stripe integration service (`src/mrm/royalties/payments.py`):
```python
class StripePaymentService:
    def __init__(self):
        stripe.api_key = settings.stripe_secret_key

    async def create_connected_account(self, party: Party) -> str:
        """Create a Stripe Connect Express account for a rights holder."""
        account = stripe.Account.create(
            type="express",
            country=party.country_code or "US",
            email=party.contact_info.get("email"),
            capabilities={"transfers": {"requested": True}},
            metadata={"party_id": str(party.id)},
        )
        return account.id

    async def create_transfer(self, stripe_account_id: str, amount: Decimal, currency: str, description: str) -> str:
        """Transfer funds to a connected account."""
        transfer = stripe.Transfer.create(
            amount=int(amount * 100),  # Stripe uses cents
            currency=currency.lower(),
            destination=stripe_account_id,
            description=description,
        )
        return transfer.id
```

Party onboarding: `/catalog/parties/{id}/stripe-connect` → generates Stripe onboarding link.

**Testing**:
- `Integration (mocked Stripe): create_connected_account → returns account ID`
- `Integration (mocked Stripe): create_transfer → returns transfer ID`
- `Integration (mocked Stripe): transfer failure → raises with error code`

---

#### 10.2 — Payment Run Workflow

**What**: Create, approve, and execute payment runs that disburse calculated royalties.

**Design**:

PaymentRun model:
```python
class PaymentRun(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "payment_runs"

    run_date: Mapped[date] = mapped_column(nullable=False)
    period_label: Mapped[str | None] = mapped_column(String(50))
    settlement_currency: Mapped[str] = mapped_column(String(3), nullable=False)
    total_amount: Mapped[Decimal] = mapped_column(Numeric(16, 4), nullable=False)
    total_payees: Mapped[int] = mapped_column(nullable=False)
    status: Mapped[str] = mapped_column(String(30), default="draft")
    approved_by: Mapped[UUID | None] = mapped_column(ForeignKey("users.id"))
    approved_at: Mapped[datetime | None]
    completed_at: Mapped[datetime | None]
    created_by: Mapped[UUID | None] = mapped_column(ForeignKey("users.id"))
    summary: Mapped[dict] = mapped_column(JSONB, default=dict)

class PaymentRunItem(Base, UUIDMixin):
    __tablename__ = "payment_run_items"

    payment_run_id: Mapped[UUID] = mapped_column(ForeignKey("payment_runs.id", ondelete="CASCADE"))
    party_id: Mapped[UUID] = mapped_column(ForeignKey("parties.id"))
    total_amount: Mapped[Decimal] = mapped_column(Numeric(14, 4), nullable=False)
    currency_code: Mapped[str] = mapped_column(String(3), nullable=False)
    payment_method: Mapped[str | None] = mapped_column(String(30))
    status: Mapped[str] = mapped_column(String(30), default="pending")
    sent_at: Mapped[datetime | None]
    confirmed_at: Mapped[datetime | None]
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    payment_details: Mapped[dict] = mapped_column(JSONB, default=dict)
    breakdown: Mapped[dict] = mapped_column(JSONB, default=dict)
```

Payment run status lifecycle: `draft` → `approved` → `processing` → `completed` | `failed`.
Payment item status: `pending` → `processing` → `sent` → `confirmed` | `failed` | `returned`.

API endpoints:
| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| POST | /payments/runs | {period_label, settlement_currency, batch_ids} | PaymentRunResponse | accounting+ |
| GET | /payments/runs | status, period | PaginatedResponse[PaymentRunResponse] | viewer+ |
| GET | /payments/runs/{id} | — | PaymentRunDetailResponse (includes items) | viewer+ |
| POST | /payments/runs/{id}/approve | — | PaymentRunResponse | manager+ |
| POST | /payments/runs/{id}/execute | — | PaymentRunResponse (202, async) | accounting+ |
| GET | /payments/runs/{id}/items | — | PaginatedResponse[PaymentRunItemResponse] | viewer+ |

Payment run creation aggregates all unpaid royalty calculations for each party into PaymentRunItems. Execution dispatches a Celery task that processes each item via Stripe Connect.

**Testing**:
- `Integration: POST payment run → aggregates royalties per party correctly`
- `Integration: approve → status changes, approved_by set`
- `Integration: execute without approval → 400 Bad Request`
- `Integration (mocked Stripe): execute → items move to sent → confirmed`
- `Integration (mocked Stripe): payment failure → item status=failed, error stored`
- `Integration: GET run detail → includes per-item breakdown`

---

#### 10.3 — Multi-Currency Disbursement & Payment Tracking

**What**: Handle payments in multiple currencies based on party preferences, with full payment history.

**Design**:

Currency selection logic:
```python
async def determine_payment_currency(party: Party) -> str:
    """Determine payment currency from party's bank account or preferences."""
    prefs = party.contact_info.get("payment_preferences", {})
    accounts = party.contact_info.get("bank_accounts", [])
    primary = next((a for a in accounts if a.get("is_primary")), None)
    if primary:
        return primary.get("currency", "USD")
    return prefs.get("preferred_currency", "USD")
```

If payment currency differs from settlement currency, apply FX conversion at payment time.

Payment history endpoint for rights holders:
| Method | Path | Query | Response | Auth |
|--------|------|-------|----------|------|
| GET | /payments/history | party_id, from_date, to_date | PaginatedResponse[PaymentHistoryResponse] | writer+ |

```python
class PaymentHistoryResponse(BaseModel):
    payment_date: date
    period_label: str
    total_amount: Decimal
    currency: str
    payment_method: str
    status: str
    breakdown: dict  # by work, by rights type, by territory
```

**Testing**:
- `Unit: determine_payment_currency with GBP primary account → "GBP"`
- `Unit: determine_payment_currency with no accounts → "USD" default`
- `Integration: payment in GBP for USD settlement → FX conversion applied`
- `Integration: GET /payments/history → returns chronological payment list for party`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation               ─── required by everything
    │
Phase 2: Catalog & Parties        ─── requires Phase 1
    │
Phase 3: Rights & Splits          ─── requires Phase 2
    │
Phase 4: Contract Management      ─── requires Phase 2
    │
Phase 5: Statement Ingestion      ─── requires Phase 2
    │
Phase 6: Royalty Calculation       ─── requires Phase 3 + Phase 5
    │                                  (Phase 4 needed for deductions in 6.3)
Phase 7: PRO Registration         ─── requires Phase 3
    │
Phase 8: Frontend Application     ─── requires Phases 2-7 (progressive)
    │                                  8.1-8.2 can start after Phase 2
    │                                  8.3 after Phase 3-4
    │                                  8.4 after Phase 5-7
    │
Phase 9: Audit Trail & Reporting  ─── requires Phase 6
    │
Phase 10: Payment Disbursement    ─── requires Phase 6

Parallelism opportunities:
  - Phases 3, 4, and 5 can be developed concurrently after Phase 2
  - Phase 7 can be developed concurrently with Phase 6 (both need Phase 3)
  - Phase 8 (frontend) can start 8.1-8.2 in parallel with Phases 3-5
  - Phases 9 and 10 can be developed concurrently after Phase 6
```

---

## Definition of Done (per phase)

1. All tasks implemented with complete Design and Testing sections satisfied.
2. All unit tests pass (`pytest tests/unit/ --tb=short`).
3. All integration tests pass (`pytest tests/integration/ --tb=short`).
4. Ruff linting passes with zero errors (`ruff check src/`).
5. mypy type checking passes in strict mode (`mypy src/mrm/`).
6. Docker build succeeds (`docker compose build`).
7. All services start and pass health checks (`docker compose up` → all healthy within 60s).
8. Alembic migrations run cleanly (`alembic upgrade head` from empty DB).
9. New API endpoints appear in the auto-generated OpenAPI spec at `/docs`.
10. New configuration options documented in `.env.example`.
11. E2E workflow test passes for the phase's primary use case.

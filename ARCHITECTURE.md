# ARCHITECTURE.md

## Стек

| Слой | Технология | Версия |
|------|-----------|--------|
| Backend | Python + FastAPI | 3.12 / 0.115+ |
| ORM | SQLAlchemy async + asyncpg | 2.0 |
| Миграции | Alembic | latest |
| AI | Anthropic SDK | latest |
| Excel | openpyxl | 3.1.5 |
| PDF | **fpdf2** (не weasyprint!) | 2.8+ |
| Rate limiting | slowapi | latest |
| Логирование | structlog | latest |
| Frontend | React 18 + TypeScript | 18.3 / 5.6 |
| Сборка | Vite | 5.4 |
| State | Zustand | 5.0 |
| HTTP | Axios | 1.7.9 |
| БД | PostgreSQL | 15+ |
| Деплой | Render.com | — |

> **Важно:** fpdf2, не weasyprint. weasyprint требует libcairo/libpango —
> они не работают на Render.com без кастомного Docker-образа.

---

## Структура файлов

```
smeta-ai/
├── app/
│   ├── main.py              # FastAPI app, CORS, lifespan, роутеры
│   ├── config.py            # Settings через pydantic-settings
│   ├── database.py          # AsyncEngine, SessionLocal, get_db
│   ├── auth.py              # bcrypt + JWT helpers
│   ├── models/
│   │   ├── __init__.py      # импорт всех моделей для Alembic
│   │   ├── user.py
│   │   ├── project.py
│   │   ├── task.py
│   │   ├── task_input_file.py
│   │   ├── task_result.py
│   │   ├── task_version.py
│   │   ├── estimate_item.py
│   │   └── price.py
│   ├── schemas/
│   │   ├── auth.py
│   │   ├── task.py
│   │   ├── project.py
│   │   ├── estimate.py
│   │   └── admin.py
│   ├── routers/
│   │   ├── auth.py
│   │   ├── tasks.py
│   │   ├── projects.py
│   │   └── admin.py
│   └── services/
│       ├── claude_service.py
│       ├── task_processor.py
│       ├── excel_service.py
│       ├── pdf_service.py
│       ├── price_service.py
│       ├── snapshot_service.py
│       ├── optimization_service.py
│       └── analogue_service.py
├── alembic/
│   ├── env.py
│   └── versions/
│       └── 001_initial.py
├── frontend/
│   ├── src/
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── api/
│   │   │   └── client.ts        # Axios instance + JWT interceptor
│   │   ├── store/
│   │   │   └── auth.ts          # Zustand: token, role
│   │   ├── pages/
│   │   │   ├── Login.tsx
│   │   │   ├── TaskCreate.tsx
│   │   │   ├── TaskStatus.tsx
│   │   │   ├── EstimateView.tsx
│   │   │   └── Admin.tsx
│   │   └── components/
│   │       ├── Layout.tsx
│   │       ├── ProjectsSidebar.tsx
│   │       ├── TaskTypeSelector.tsx
│   │       ├── FileUpload.tsx
│   │       ├── StatusBadge.tsx
│   │       ├── VersionHistoryDrawer.tsx
│   │       ├── OptimizationChecklist.tsx
│   │       ├── AnaloguePanel.tsx
│   │       └── ProtectedRoute.tsx
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts
├── requirements.txt
├── alembic.ini
├── .env.example
├── render.yaml
└── Dockerfile
```

---

## app/config.py

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    anthropic_api_key: str
    jwt_secret: str
    jwt_expire_hours: int = 24
    user_password: str
    admin_password: str
    max_files_per_request: int = 5
    max_file_size_mb: int = 50
    search_city: str = "Екатеринбург"
    vat_rate: float = 20.0

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## app/database.py

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase
from app.config import settings

engine = create_async_engine(
    settings.database_url,
    pool_size=5, max_overflow=10, pool_recycle=300
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

async def get_db():
    async with SessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

---

## app/main.py

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.database import engine
from app.models import Base  # импортирует все модели
from app.services.price_service import price_service
from app.routers import auth, tasks, projects, admin

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Старт: прогреть кэш прайсов
    await price_service.load_cache()
    yield
    await engine.dispose()

app = FastAPI(title="Smeta AI", lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router, prefix="/auth", tags=["auth"])
app.include_router(tasks.router, prefix="/tasks", tags=["tasks"])
app.include_router(projects.router, prefix="/projects", tags=["projects"])
app.include_router(admin.router, prefix="/admin", tags=["admin"])

@app.get("/health")
async def health():
    return {"status": "ok"}
```

---

## requirements.txt

```
fastapi==0.115.5
uvicorn[standard]
sqlalchemy[asyncio]==2.0.*
asyncpg
alembic
anthropic
openpyxl==3.1.5
fpdf2>=2.8.0
python-jose[cryptography]
passlib[bcrypt]
python-multipart
slowapi
structlog
pydantic-settings
```

---

## render.yaml

```yaml
services:
  - type: web
    name: smeta-ai-backend
    env: python
    buildCommand: pip install -r requirements.txt
    startCommand: alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port $PORT
    envVars:
      - key: DATABASE_URL
        fromDatabase:
          name: smeta-ai-db
          property: connectionString

  - type: web
    name: smeta-ai-frontend
    env: static
    buildCommand: cd frontend && npm install && npm run build && cp dist/index.html dist/404.html
    staticPublishPath: ./frontend/dist

databases:
  - name: smeta-ai-db
    plan: free
```

---

## Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["sh", "-c", "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000"]
```

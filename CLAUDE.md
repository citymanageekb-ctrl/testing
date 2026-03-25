# CLAUDE.md — Smeta AI

Ты разрабатываешь **Smeta AI** — веб-приложение для автоматизации строительных смет на базе Claude API.

Читай файлы в порядке:
1. `docs/ARCHITECTURE.md` — структура проекта, стек, правила
2. `docs/MODELS.md` — все таблицы БД
3. `docs/API.md` — все эндпоинты
4. `docs/SERVICES.md` — бизнес-логика сервисов
5. `docs/FRONTEND.md` — страницы и компоненты

---

## Правила разработки (обязательно соблюдать)

### Общие
- Пиши **рабочий код**, не заглушки. Каждый файл должен быть полным.
- Не добавляй функциональность, которой нет в ТЗ.
- Комментарии только там, где логика неочевидна.
- Никаких `TODO`, `pass`, `...` в финальном коде.

### Backend
- Async везде: все функции БД через `async/await` + `asyncpg`.
- Pydantic v2 для схем (не v1).
- Dependency injection через FastAPI `Depends()`.
- Ошибки — всегда `HTTPException` с понятным `detail`.
- Логирование через `structlog`, не `print`.

### Frontend
- TypeScript строго: `strict: true` в tsconfig, никаких `any`.
- Компоненты — функциональные, хуки для состояния.
- Все API-вызовы через `src/api/client.ts` (Axios instance).
- Никакого прямого `fetch()` в компонентах.

### Безопасность
- Пароли только из env, никогда в коде.
- JWT проверяется в каждом защищённом запросе.
- Входные файлы: проверять mime_type И размер.

---

## Порядок разработки

Реализуй в этом порядке — каждый шаг должен быть рабочим:

```
1. Инфраструктура
   ├── requirements.txt, .env.example
   ├── app/config.py (pydantic-settings)
   ├── app/database.py (AsyncEngine, get_db)
   └── alembic/ (начальная настройка)

2. Модели + миграция
   ├── app/models/ (все 9 таблиц)
   └── alembic/versions/001_initial.py

3. Auth
   ├── app/auth.py (bcrypt + JWT)
   └── app/routers/auth.py

4. Роутеры (тонкий слой, только валидация → сервис)
   ├── app/routers/tasks.py
   ├── app/routers/projects.py
   └── app/routers/admin.py

5. Сервисы (основная логика)
   ├── app/services/claude_service.py
   ├── app/services/price_service.py
   ├── app/services/task_processor.py
   ├── app/services/excel_service.py
   ├── app/services/pdf_service.py
   ├── app/services/snapshot_service.py
   ├── app/services/optimization_service.py
   └── app/services/analogue_service.py

6. app/main.py (всё собрать, lifespan, CORS)

7. Frontend
   ├── Axios client + Zustand store
   ├── ProtectedRoute
   ├── Страницы: Login → TaskCreate → TaskStatus → EstimateView → Admin
   └── Компоненты (Layout, Sidebar, FileUpload, StatusBadge и др.)

8. Конфигурация деплоя
   ├── render.yaml
   └── Dockerfile
```

---

## Ключевые переменные окружения

```env
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/smeta_ai
ANTHROPIC_API_KEY=sk-ant-...
JWT_SECRET=your-256-bit-secret
JWT_EXPIRE_HOURS=24
USER_PASSWORD=...
ADMIN_PASSWORD=...
MAX_FILES_PER_REQUEST=5
MAX_FILE_SIZE_MB=50
SEARCH_CITY=Екатеринбург
VAT_RATE=20
```

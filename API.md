# API.md — Эндпоинты

## Общее

- Все эндпоинты кроме `/auth/login` и `/health` требуют `Authorization: Bearer {JWT}`.
- Роутеры — **тонкий слой**: принять запрос → вызвать сервис → вернуть ответ.
- Ошибки: `HTTPException` с понятным `detail`.
- Пагинация: `page` (default 1) + `page_size` (default 20, max 100).

---

## Auth

### POST /auth/login

```python
# Запрос
class LoginRequest(BaseModel):
    password: str

# Ответ
class LoginResponse(BaseModel):
    access_token: str
    role: str          # 'user' | 'admin'
    expires_in: int    # 86400 (секунды)
```

Логика: сравнить password с `settings.user_password` и `settings.admin_password` через bcrypt.
Вернуть JWT с payload `{sub: "1" | "2", role: "user" | "admin", exp: ...}`.
Пользователи не хранятся в БД — только два фиксированных пароля из env.

> **Примечание**: Это однопользовательская система с двумя ролями. user_id в tasks = 1 (user) или 2 (admin).

---

## Tasks

Зависимость: `current_user = Depends(get_current_user)` — любой авторизованный.

### POST /tasks

`multipart/form-data`:
- `task_type: str` — обязательный, из TASK_TYPES
- `prompt: str` — опциональный
- `files: list[UploadFile]` — до `settings.max_files_per_request` файлов, каждый до `settings.max_file_size_mb` МБ

Валидация:
- task_type в TASK_TYPES, иначе 422
- Количество файлов ≤ max_files_per_request, иначе 400
- Размер каждого файла ≤ max_file_size_mb * 1024 * 1024, иначе 400
- mime_type файла в ALLOWED_MIME_TYPES[task_type], иначе 400

Логика:
1. Создать Task (status=pending, estimate_status=uploaded)
2. Сохранить файлы в task_input_files
3. `asyncio.create_task(task_processor.process(task.id))` — фоновое выполнение
4. Вернуть `{"task_id": str(task.id)}`

### GET /tasks/{task_id}/status

```python
class TaskStatusResponse(BaseModel):
    id: str
    task_type: str
    status: str
    progress_message: str | None
    error_message: str | None
    estimate_status: str | None
    created_at: datetime
    updated_at: datetime
```

### POST /tasks/{task_id}/cancel

Если status в ('pending', 'processing') → status = 'cancelled'. Иначе 400.

### POST /tasks/{task_id}/message

```python
class MessageRequest(BaseModel):
    content: str
```

Логика:
1. Добавить `{"role": "user", "content": content}` в task.chat_history
2. Сбросить error_message = None
3. task.status = "pending"
4. `asyncio.create_task(task_processor.process(task.id))`
5. Вернуть `{"ok": True}`

### GET /tasks/{task_id}/results

```python
class TaskResultFile(BaseModel):
    id: int
    file_name: str
    mime_type: str
    created_at: datetime
```
Вернуть список TaskResultFile для task_id.

### GET /results/{file_id}/download

Вернуть `FileResponse` или `Response` с:
- `Content-Disposition: attachment; filename="{file_name}"`
- `Content-Type: {mime_type}`
- body: file_data

---

## Projects

Зависимость: `current_user = Depends(get_current_user)`.

### POST /projects
Body: `{"name": str, "description": str | None}`
Создать Project с user_id = current_user.id.

### GET /projects
Вернуть список проектов current_user, отсортированных по updated_at DESC.

### GET /projects/{project_id}
Вернуть проект + список задач в нём (tasks где project_id = project_id).

### PUT /projects/{project_id}
Body: `{"name": str, "description": str | None}`
Обновить поля. Проверить что project.user_id == current_user.id.

### DELETE /projects/{project_id}
Удалить проект. Задачи остаются (project_id → NULL через ON DELETE SET NULL).

### POST /projects/{project_id}/estimates/{task_id}
Привязать задачу к проекту: `task.project_id = project_id`.

### DELETE /projects/{project_id}/estimates/{task_id}
Отвязать задачу от проекта: `task.project_id = None`.

### PATCH /projects/estimates/{task_id}/status
```python
class EstimateStatusUpdate(BaseModel):
    status: str   # uploaded | calculated | optimized
    updated_by: str  # manual | auto
```

### GET /projects/estimates/{task_id}/items
Вернуть все EstimateItem для task_id + `vat_rate: settings.vat_rate` в ответе.

```python
class EstimateItemsResponse(BaseModel):
    items: list[EstimateItemSchema]
    vat_rate: float
    total_work: float    # сумма work_price * quantity
    total_mat: float     # сумма mat_price * quantity
    total: float         # total_work + total_mat
    total_vat: float     # total * (1 + vat_rate/100)
```

### GET /projects/estimates/{task_id}/versions
Список TaskVersion для task_id, отсортированных по version_number DESC.

### POST /projects/estimates/{task_id}/versions/{version_id}/restore
Вызвать `snapshot_service.restore_snapshot(task_id, version_id)`.

### POST /projects/estimates/{task_id}/optimize/plan
Вызвать `optimization_service.get_optimization_plan(task_id)`.
Вернуть список позиций для оптимизации + ожидаемую экономию.

### POST /projects/estimates/{task_id}/optimize/execute
```python
class OptimizeExecuteRequest(BaseModel):
    item_ids: list[str]  # UUID позиций для оптимизации
```
Вызвать `optimization_service.execute_optimization(task_id, item_ids)`.

### POST /projects/estimates/{task_id}/items/{item_id}/find-analogues
Вызвать `analogue_service.find_analogues(task_id, item_id)`.
Вернуть список аналогов.

### POST /projects/estimates/{task_id}/items/{item_id}/apply-analogue
```python
class ApplyAnalogueRequest(BaseModel):
    name: str
    price: float
    unit: str
    supplier: str
    source_url: str | None
    analogue_note: str
```
Вызвать `analogue_service.apply_analogue(item_id, data)`.

### POST /projects/estimates/{task_id}/items/{item_id}/revert-analogue
Вызвать `analogue_service.revert_analogue(item_id)`.

---

## Admin

Зависимость: `admin_user = Depends(get_admin_user)` — только role='admin'.

### GET /admin/tasks
Query params: `status`, `task_type`, `date_from`, `date_to`, `page`, `page_size`
Вернуть все задачи с пагинацией.

### GET /admin/tasks/{task_id}
Полные детали задачи включая chat_history.

### DELETE /admin/tasks/{task_id}
Удалить задачу (CASCADE удалит input_files, results, versions, items).

### GET /admin/tasks/{task_id}/download-input/{file_index}
Скачать входной файл по file_index из task_input_files.

### GET /admin/price-lists/info
```python
class PriceListInfo(BaseModel):
    works: dict | None    # {filename, updated_at} или None если не загружен
    materials: dict | None
```

### POST /admin/price-lists/works
`multipart/form-data`: `file: UploadFile` (XLSX, до 20 МБ)
Логика:
1. Сохранить в price_lists (type='works')
2. Распарсить XLSX → bulk upsert в price_works
3. `price_service.reload_cache()`

### POST /admin/price-lists/materials
Аналогично для materials → price_materials.

---

## app/auth.py

```python
from datetime import datetime, timedelta, timezone
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from app.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
bearer = HTTPBearer()

def create_access_token(user_id: int, role: str) -> str:
    expire = datetime.now(timezone.utc) + timedelta(hours=settings.jwt_expire_hours)
    return jwt.encode(
        {"sub": str(user_id), "role": role, "exp": expire},
        settings.jwt_secret, algorithm="HS256"
    )

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

class CurrentUser:
    def __init__(self, user_id: int, role: str):
        self.id = user_id
        self.role = role

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(bearer)
) -> CurrentUser:
    try:
        payload = jwt.decode(credentials.credentials, settings.jwt_secret, algorithms=["HS256"])
        return CurrentUser(user_id=int(payload["sub"]), role=payload["role"])
    except JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")

async def get_admin_user(current_user: CurrentUser = Depends(get_current_user)) -> CurrentUser:
    if current_user.role != "admin":
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Admin only")
    return current_user
```

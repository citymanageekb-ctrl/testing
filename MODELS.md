# MODELS.md — Модели данных

Все модели наследуют от `app.database.Base`.
UUID генерируется через `default=uuid.uuid4`.
DateTime — всегда timezone-aware (`timezone=True`).

---

## users

```python
# app/models/user.py
import uuid
from datetime import datetime, timezone
from sqlalchemy import Integer, String, DateTime
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.database import Base

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    role: Mapped[str] = mapped_column(String(10), nullable=False)  # 'user' | 'admin'
    password_hash: Mapped[str] = mapped_column(String(255), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )

    tasks: Mapped[list["Task"]] = relationship(back_populates="user")
    projects: Mapped[list["Project"]] = relationship(back_populates="user")
```

---

## projects

```python
# app/models/project.py
import uuid
from datetime import datetime, timezone
from sqlalchemy import String, Text, Integer, DateTime, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.database import Base

class Project(Base):
    __tablename__ = "projects"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    user_id: Mapped[int | None] = mapped_column(
        Integer, ForeignKey("users.id", ondelete="SET NULL"), nullable=True
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc)
    )

    user: Mapped["User"] = relationship(back_populates="projects")
    tasks: Mapped[list["Task"]] = relationship(back_populates="project")
```

---

## tasks

> Поля `user_role`, `input_files`, `input_file_data` удалены.
> Файлы хранятся в `task_input_files`. Роль берётся через JOIN с users.

```python
# app/models/task.py
import uuid
from datetime import datetime, timezone
from sqlalchemy import String, Text, Integer, DateTime, ForeignKey
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.database import Base

class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    task_type: Mapped[str] = mapped_column(String(50), nullable=False)
    status: Mapped[str] = mapped_column(String(20), nullable=False, default="pending")
    user_prompt: Mapped[str | None] = mapped_column(Text, nullable=True)
    chat_history: Mapped[list] = mapped_column(JSONB, nullable=False, default=list)
    progress_message: Mapped[str | None] = mapped_column(String(500), nullable=True)
    error_message: Mapped[str | None] = mapped_column(Text, nullable=True)
    project_id: Mapped[uuid.UUID | None] = mapped_column(
        UUID(as_uuid=True), ForeignKey("projects.id", ondelete="SET NULL"), nullable=True
    )
    user_id: Mapped[int | None] = mapped_column(
        Integer, ForeignKey("users.id", ondelete="SET NULL"), nullable=True
    )
    estimate_status: Mapped[str | None] = mapped_column(String(50), nullable=True)
    estimate_status_updated_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    estimate_status_updated_by: Mapped[str | None] = mapped_column(String(20), nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc)
    )

    user: Mapped["User"] = relationship(back_populates="tasks")
    project: Mapped["Project"] = relationship(back_populates="tasks")
    input_files: Mapped[list["TaskInputFile"]] = relationship(
        back_populates="task", cascade="all, delete-orphan", order_by="TaskInputFile.file_index"
    )
    results: Mapped[list["TaskResult"]] = relationship(
        back_populates="task", cascade="all, delete-orphan"
    )
    versions: Mapped[list["TaskVersion"]] = relationship(
        back_populates="task", cascade="all, delete-orphan"
    )
    estimate_items: Mapped[list["EstimateItem"]] = relationship(
        back_populates="task", cascade="all, delete-orphan", order_by="EstimateItem.position"
    )
```

### Допустимые значения task_type

```python
TASK_TYPES = {
    "LIST_FROM_TZ",
    "LIST_FROM_TZ_PROJECT",
    "LIST_FROM_PROJECT",
    "RESEARCH_PROJECT",
    "SMETA_FROM_LIST",
    "SMETA_FROM_TZ",
    "SMETA_FROM_TZ_PROJECT",
    "SMETA_FROM_PROJECT",
    "SMETA_FROM_EDC_PROJECT",
    "SMETA_FROM_GRAND_PROJECT",
    "SCAN_TO_EXCEL",
    "COMPARE_PROJECT_SMETA",
}

# Допустимые входные форматы по типу задачи
ALLOWED_MIME_TYPES = {
    "LIST_FROM_TZ": ["application/pdf", "image/jpeg", "image/png"],
    "LIST_FROM_TZ_PROJECT": ["application/pdf", "image/jpeg", "image/png"],
    "LIST_FROM_PROJECT": ["application/pdf", "image/jpeg", "image/png"],
    "RESEARCH_PROJECT": ["application/pdf", "image/jpeg", "image/png"],
    "SMETA_FROM_LIST": [
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        "application/vnd.ms-excel",
    ],
    "SMETA_FROM_TZ": ["application/pdf", "image/jpeg", "image/png"],
    "SMETA_FROM_TZ_PROJECT": ["application/pdf", "image/jpeg", "image/png"],
    "SMETA_FROM_PROJECT": ["application/pdf", "image/jpeg", "image/png"],
    "SMETA_FROM_EDC_PROJECT": ["application/pdf", "text/xml", "application/xml"],
    "SMETA_FROM_GRAND_PROJECT": ["application/pdf", "text/xml", "application/xml"],
    "SCAN_TO_EXCEL": ["application/pdf", "image/jpeg", "image/png"],
    "COMPARE_PROJECT_SMETA": [
        "application/pdf",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    ],
}
```

---

## task_input_files

```python
# app/models/task_input_file.py
from datetime import datetime, timezone
from sqlalchemy import Integer, String, LargeBinary, DateTime, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.database import Base

class TaskInputFile(Base):
    __tablename__ = "task_input_files"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    task_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("tasks.id", ondelete="CASCADE"), nullable=False
    )
    file_index: Mapped[int] = mapped_column(Integer, nullable=False)  # 0-based
    file_name: Mapped[str] = mapped_column(String(255), nullable=False)
    mime_type: Mapped[str] = mapped_column(String(100), nullable=False)
    file_data: Mapped[bytes] = mapped_column(LargeBinary, nullable=False)
    size_bytes: Mapped[int] = mapped_column(Integer, nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )

    task: Mapped["Task"] = relationship(back_populates="input_files")
```

---

## task_results

```python
# app/models/task_result.py
from datetime import datetime, timezone
from sqlalchemy import Integer, String, LargeBinary, DateTime, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.database import Base

class TaskResult(Base):
    __tablename__ = "task_results"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    task_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("tasks.id", ondelete="CASCADE"), nullable=False
    )
    file_name: Mapped[str] = mapped_column(String(255), nullable=False)
    mime_type: Mapped[str] = mapped_column(String(100), nullable=False)
    file_data: Mapped[bytes] = mapped_column(LargeBinary, nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )

    task: Mapped["Task"] = relationship(back_populates="results")
```

---

## task_versions

```python
# app/models/task_version.py
import uuid
from datetime import datetime, timezone
from sqlalchemy import Integer, String, Text, DateTime, ForeignKey
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.database import Base

class TaskVersion(Base):
    __tablename__ = "task_versions"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    task_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("tasks.id", ondelete="CASCADE"), nullable=False
    )
    version_number: Mapped[int] = mapped_column(Integer, nullable=False)
    # snapshot = {"items": [{все поля EstimateItem как dict}]}
    snapshot: Mapped[dict] = mapped_column(JSONB, nullable=False)
    change_description: Mapped[str | None] = mapped_column(Text, nullable=True)
    # change_type: optimization | manual_edit | restoration
    change_type: Mapped[str] = mapped_column(String(50), nullable=False)
    # created_by: user | system
    created_by: Mapped[str] = mapped_column(String(20), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )

    task: Mapped["Task"] = relationship(back_populates="versions")
```

---

## estimate_items

```python
# app/models/estimate_item.py
import uuid
from datetime import datetime, timezone
from sqlalchemy import Integer, String, Text, Float, Boolean, DateTime, ForeignKey
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.database import Base

class EstimateItem(Base):
    __tablename__ = "estimate_items"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    task_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("tasks.id", ondelete="CASCADE"), nullable=False
    )
    position: Mapped[int] = mapped_column(Integer, nullable=False)
    type: Mapped[str] = mapped_column(String(20), nullable=False)   # 'Работа' | 'Материал'
    name: Mapped[str] = mapped_column(Text, nullable=False)
    unit: Mapped[str] = mapped_column(String(100), nullable=False)
    quantity: Mapped[float] = mapped_column(Float, nullable=False)
    work_price: Mapped[float] = mapped_column(Float, nullable=False, default=0.0)
    mat_price: Mapped[float] = mapped_column(Float, nullable=False, default=0.0)
    section: Mapped[str | None] = mapped_column(Text, nullable=True)
    notes: Mapped[str | None] = mapped_column(Text, nullable=True)
    is_analogue: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False)
    original_item_id: Mapped[uuid.UUID | None] = mapped_column(
        UUID(as_uuid=True), ForeignKey("estimate_items.id", ondelete="SET NULL"), nullable=True
    )
    analogue_note: Mapped[str | None] = mapped_column(Text, nullable=True)
    # extra = {"price_list_name": str, "sources": [...], "analogues_cache": [...]}
    extra: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc)
    )

    task: Mapped["Task"] = relationship(back_populates="estimate_items")
```

---

## price_works, price_materials, price_lists

```python
# app/models/price.py
from datetime import datetime, timezone
from sqlalchemy import Integer, String, Text, Float, DateTime, LargeBinary
from sqlalchemy.dialects.postgresql import JSON
from sqlalchemy.orm import Mapped, mapped_column
from app.database import Base

class PriceWork(Base):
    __tablename__ = "price_works"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(Text, nullable=False)
    unit: Mapped[str] = mapped_column(String(50), nullable=False)
    # {"подрядчик": цена}
    prices: Mapped[dict] = mapped_column(JSON, nullable=False, default=dict)
    min_price: Mapped[float] = mapped_column(Float, nullable=False)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )

class PriceMaterial(Base):
    __tablename__ = "price_materials"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(Text, nullable=False)
    unit: Mapped[str] = mapped_column(String(50), nullable=False)
    price: Mapped[float] = mapped_column(Float, nullable=False)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )

class PriceList(Base):
    __tablename__ = "price_lists"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    type: Mapped[str] = mapped_column(String(20), nullable=False)  # 'works' | 'materials'
    filename: Mapped[str] = mapped_column(String(255), nullable=False)
    content: Mapped[bytes] = mapped_column(LargeBinary, nullable=False)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )
```

---

## app/models/\_\_init\_\_.py

```python
from app.database import Base
from app.models.user import User
from app.models.project import Project
from app.models.task import Task
from app.models.task_input_file import TaskInputFile
from app.models.task_result import TaskResult
from app.models.task_version import TaskVersion
from app.models.estimate_item import EstimateItem
from app.models.price import PriceWork, PriceMaterial, PriceList

__all__ = [
    "Base", "User", "Project", "Task", "TaskInputFile",
    "TaskResult", "TaskVersion", "EstimateItem",
    "PriceWork", "PriceMaterial", "PriceList",
]
```

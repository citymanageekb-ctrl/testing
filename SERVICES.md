# SERVICES.md — Сервисы

---

## ClaudeService

```python
# app/services/claude_service.py
import asyncio
import anthropic
from app.config import settings

# max_tokens по типу задачи
MAX_TOKENS = {
    "LIST_FROM_TZ": 8000,
    "LIST_FROM_TZ_PROJECT": 8000,
    "LIST_FROM_PROJECT": 8000,
    "RESEARCH_PROJECT": 8000,
    "SCAN_TO_EXCEL": 4000,
    "SMETA_FROM_LIST": 16000,
    "SMETA_FROM_TZ": 16000,
    "SMETA_FROM_TZ_PROJECT": 16000,
    "SMETA_FROM_PROJECT": 16000,
    "SMETA_FROM_EDC_PROJECT": 16000,
    "SMETA_FROM_GRAND_PROJECT": 16000,
    "COMPARE_PROJECT_SMETA": 16000,
}

class ClaudeService:
    def __init__(self):
        self.client = anthropic.AsyncAnthropic(api_key=settings.anthropic_api_key)

    async def complete(
        self,
        messages: list[dict],
        task_type: str,
        system: str = "",
        use_web_search: bool = False,
    ) -> str:
        """
        Вызвать Claude с retry + exponential backoff.
        Возвращает текст ответа.
        """
        max_tokens = MAX_TOKENS.get(task_type, 16000)
        tools = [{"type": "web_search_20250305", "name": "web_search"}] if use_web_search else []

        delays = [60, 120, 240]
        last_error = None

        for attempt, delay in enumerate([0] + delays):
            if delay:
                await asyncio.sleep(delay)
            try:
                kwargs = dict(
                    model="claude-sonnet-4-6",
                    max_tokens=max_tokens,
                    temperature=0.1,
                    messages=messages,
                )
                if system:
                    kwargs["system"] = system
                if tools:
                    kwargs["tools"] = tools

                response = await self.client.messages.create(**kwargs)

                # Собрать текст из всех text-блоков
                return "".join(
                    block.text for block in response.content
                    if hasattr(block, "text")
                )

            except anthropic.RateLimitError as e:
                last_error = e
                if attempt == len(delays):
                    raise
            except anthropic.APIStatusError as e:
                if e.status_code == 529:  # overloaded
                    last_error = e
                    if attempt == len(delays):
                        raise
                else:
                    raise

        raise last_error

claude_service = ClaudeService()
```

---

## TaskProcessor

```python
# app/services/task_processor.py
"""
Оркестратор обработки задачи. Запускается в фоне через asyncio.create_task().

Шаги:
1. task.status = 'processing'
2. Загрузить task_input_files из БД
3. Декодировать файлы (PDF → текст, XLSX → данные, изображения → base64)
4. Сформировать system prompt по task_type
5. Вызвать Claude
6. Распарсить JSON-ответ в список позиций
7. Обогатить позиции ценами через PriceService
8. Сохранить estimate_items в БД (bulk insert)
9. Сгенерировать Excel или PDF
10. Сохранить файл в task_results
11. task.status = 'completed', estimate_status = 'calculated'
"""

SYSTEM_PROMPTS = {
    "LIST_FROM_TZ": """
Ты — эксперт-сметчик. Проанализируй техническое задание и составь подробный перечень работ и материалов.

Ответь ТОЛЬКО валидным JSON без markdown-обёртки:
{
  "items": [
    {
      "type": "Работа" | "Материал",
      "name": "Наименование",
      "unit": "ед.изм.",
      "quantity": число,
      "section": "Раздел/этап или null",
      "notes": "Примечания или null"
    }
  ]
}
""",

    "SMETA_FROM_TZ": """
Ты — эксперт-сметчик. По техническому заданию составь детальную смету строительных работ.

Ответь ТОЛЬКО валидным JSON без markdown-обёртки:
{
  "items": [
    {
      "type": "Работа" | "Материал",
      "name": "Наименование",
      "unit": "ед.изм.",
      "quantity": число,
      "work_price": число или 0,
      "mat_price": число или 0,
      "section": "Раздел/этап",
      "notes": "Примечания или null"
    }
  ]
}

Для работ заполняй work_price, для материалов — mat_price.
Если цену не знаешь — оставь 0, система обогатит ценами из прайс-листа.
""",

    "SCAN_TO_EXCEL": """
Ты — эксперт-сметчик. Извлеки данные из скана сметы.

Ответь ТОЛЬКО валидным JSON без markdown-обёртки:
{
  "items": [
    {
      "type": "Работа" | "Материал",
      "name": "Наименование",
      "unit": "ед.изм.",
      "quantity": число,
      "work_price": число или 0,
      "mat_price": число или 0,
      "section": "Раздел или null",
      "notes": null
    }
  ]
}
""",

    "COMPARE_PROJECT_SMETA": """
Ты — эксперт-сметчик. Сравни проектную документацию со сметой.

Ответь ТОЛЬКО валидным JSON без markdown-обёртки:
{
  "summary": {
    "project_total": число,
    "smeta_total": число,
    "difference": число,
    "difference_pct": число
  },
  "discrepancies": [
    {
      "name": "Наименование позиции",
      "project_value": число или null,
      "smeta_value": число или null,
      "difference_pct": число,
      "severity": "critical" | "warning" | "ok"
    }
  ],
  "missing_in_smeta": ["позиция 1", "позиция 2"],
  "missing_in_project": ["позиция 1"],
  "recommendations": ["рекомендация 1", "рекомендация 2"]
}
""",
    # Аналогично для остальных типов — структура items та же
}

# Для типов без явного промпта — использовать SMETA_FROM_TZ
DEFAULT_SYSTEM_PROMPT = SYSTEM_PROMPTS["SMETA_FROM_TZ"]
```

### Декодирование файлов

```python
async def _decode_file(input_file: TaskInputFile) -> dict:
    """
    Вернуть структуру для messages[] Claude.
    """
    if input_file.mime_type in ("image/jpeg", "image/png"):
        import base64
        return {
            "type": "image",
            "source": {
                "type": "base64",
                "media_type": input_file.mime_type,
                "data": base64.b64encode(input_file.file_data).decode(),
            }
        }

    elif input_file.mime_type == "application/pdf":
        # Извлечь текст через pypdf (добавить в requirements: pypdf>=4.0)
        import io
        from pypdf import PdfReader
        reader = PdfReader(io.BytesIO(input_file.file_data))
        text = "\n".join(page.extract_text() or "" for page in reader.pages)
        return {"type": "text", "text": f"[PDF: {input_file.file_name}]\n{text}"}

    elif input_file.mime_type in (
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        "application/vnd.ms-excel"
    ):
        import io
        from openpyxl import load_workbook
        wb = load_workbook(io.BytesIO(input_file.file_data), read_only=True)
        rows = []
        for ws in wb.worksheets:
            rows.append(f"[Лист: {ws.title}]")
            for row in ws.iter_rows(values_only=True):
                if any(c is not None for c in row):
                    rows.append("\t".join(str(c or "") for c in row))
        return {"type": "text", "text": "\n".join(rows)}

    elif input_file.mime_type in ("text/xml", "application/xml"):
        text = input_file.file_data.decode("utf-8", errors="replace")
        return {"type": "text", "text": f"[XML: {input_file.file_name}]\n{text}"}

    else:
        return {"type": "text", "text": f"[Файл: {input_file.file_name}]"}
```

### Парсинг ответа Claude

```python
import json, re

def _parse_claude_response(response_text: str) -> list[dict]:
    """
    Извлечь JSON из ответа. Claude может обернуть в ```json ... ```.
    """
    # Убрать markdown-обёртку если есть
    clean = re.sub(r"```(?:json)?\s*", "", response_text).strip()
    clean = clean.rstrip("`").strip()

    data = json.loads(clean)
    return data.get("items", [])
```

### Обогащение ценами

```python
async def _enrich_with_prices(items: list[dict]) -> list[dict]:
    from app.services.price_service import price_service
    for item in items:
        if item.get("work_price", 0) == 0 and item["type"] == "Работа":
            price = await price_service.lookup(item["name"], "work")
            if price:
                item["work_price"] = price
        if item.get("mat_price", 0) == 0 and item["type"] == "Материал":
            price = await price_service.lookup(item["name"], "material")
            if price:
                item["mat_price"] = price
    return items
```

---

## PriceService

```python
# app/services/price_service.py
"""
In-memory кэш прайсов. Загружается при старте через lifespan.
Поиск: 1) точное совпадение → 2) нечёткое (difflib) → 3) Claude fallback.
"""
import difflib
from app.config import settings

class PriceService:
    def __init__(self):
        # {normalized_name: price}
        self._works: dict[str, float] = {}
        self._materials: dict[str, float] = {}

    def _normalize(self, name: str) -> str:
        return " ".join(name.lower().strip().split())

    async def load_cache(self):
        """Загрузить из БД в memory при старте."""
        from app.database import SessionLocal
        from app.models.price import PriceWork, PriceMaterial
        from sqlalchemy import select
        async with SessionLocal() as db:
            works = (await db.execute(select(PriceWork))).scalars().all()
            self._works = {self._normalize(w.name): w.min_price for w in works}

            mats = (await db.execute(select(PriceMaterial))).scalars().all()
            self._materials = {self._normalize(m.name): m.price for m in mats}

    async def reload_cache(self):
        """Вызвать после загрузки нового прайс-листа."""
        self._works.clear()
        self._materials.clear()
        await self.load_cache()

    async def lookup(self, name: str, type_: str) -> float | None:
        """
        type_: 'work' | 'material'
        Возвращает цену или None.
        """
        cache = self._works if type_ == "work" else self._materials
        normalized = self._normalize(name)

        # 1. Точное совпадение
        if normalized in cache:
            return cache[normalized]

        # 2. Нечёткое совпадение (порог 0.8)
        matches = difflib.get_close_matches(normalized, cache.keys(), n=1, cutoff=0.8)
        if matches:
            return cache[matches[0]]

        # 3. Claude fallback с web_search
        return await self._claude_lookup(name, type_)

    async def _claude_lookup(self, name: str, type_: str) -> float | None:
        from app.services.claude_service import claude_service
        type_label = "работы" if type_ == "work" else "материала"
        prompt = (
            f"Найди актуальную рыночную цену {type_label} '{name}' "
            f"в городе {settings.search_city}. "
            f"Ответь ТОЛЬКО числом — ценой в рублях за единицу измерения. "
            f"Без пояснений, только число."
        )
        try:
            response = await claude_service.complete(
                messages=[{"role": "user", "content": prompt}],
                task_type="SMETA_FROM_TZ",
                use_web_search=True,
            )
            return float(response.strip().replace(",", ".").replace(" ", ""))
        except Exception:
            return None

price_service = PriceService()
```

---

## ExcelService

```python
# app/services/excel_service.py
"""
Генерация .xlsx.
generate_list(items) → байты файла (для LIST_* задач)
generate_smeta(items, vat_rate) → байты файла (для SMETA_* задач)
"""
import io
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side
from openpyxl.utils import get_column_letter

HEADER_FILL = PatternFill("solid", fgColor="4472C4")
TOTAL_FILL = PatternFill("solid", fgColor="D9E1F2")
HEADER_FONT = Font(bold=True, color="FFFFFF", name="Arial", size=10)
NORMAL_FONT = Font(name="Arial", size=10)
THIN = Side(style="thin", color="CCCCCC")
ALL_BORDERS = Border(left=THIN, right=THIN, top=THIN, bottom=THIN)

def _header_row(ws, columns: list[str]):
    for col, title in enumerate(columns, 1):
        cell = ws.cell(row=1, column=col, value=title)
        cell.fill = HEADER_FILL
        cell.font = HEADER_FONT
        cell.alignment = Alignment(horizontal="center", wrap_text=True)
        cell.border = ALL_BORDERS

def _auto_width(ws):
    for col in ws.columns:
        max_len = max((len(str(c.value or "")) for c in col), default=10)
        ws.column_dimensions[get_column_letter(col[0].column)].width = min(max_len + 4, 60)

def generate_list(items: list[dict]) -> bytes:
    wb = Workbook()
    # Лист 1: Перечень (все позиции)
    ws_all = wb.active
    ws_all.title = "Перечень"
    _header_row(ws_all, ["№", "Тип", "Наименование", "Ед.изм.", "Кол-во", "Раздел", "Примечания"])
    for i, item in enumerate(items, 1):
        row = [i, item.get("type"), item.get("name"), item.get("unit"),
               item.get("quantity"), item.get("section"), item.get("notes")]
        for col, val in enumerate(row, 1):
            cell = ws_all.cell(row=i+1, column=col, value=val)
            cell.font = NORMAL_FONT
            cell.border = ALL_BORDERS

    # Лист 2: Работы
    ws_w = wb.create_sheet("Работы")
    works = [it for it in items if it.get("type") == "Работа"]
    _header_row(ws_w, ["№", "Наименование", "Ед.изм.", "Кол-во"])
    for i, item in enumerate(works, 1):
        for col, val in enumerate([i, item["name"], item["unit"], item["quantity"]], 1):
            ws_w.cell(row=i+1, column=col, value=val).font = NORMAL_FONT

    # Лист 3: Материалы
    ws_m = wb.create_sheet("Материалы")
    mats = [it for it in items if it.get("type") == "Материал"]
    _header_row(ws_m, ["№", "Наименование", "Ед.изм.", "Кол-во"])
    for i, item in enumerate(mats, 1):
        for col, val in enumerate([i, item["name"], item["unit"], item["quantity"]], 1):
            ws_m.cell(row=i+1, column=col, value=val).font = NORMAL_FONT

    for ws in [ws_all, ws_w, ws_m]:
        _auto_width(ws)

    buf = io.BytesIO()
    wb.save(buf)
    return buf.getvalue()


def generate_smeta(items: list[dict], vat_rate: float = 20.0) -> bytes:
    wb = Workbook()
    ws = wb.active
    ws.title = "Смета"
    _header_row(ws, ["№", "Раздел", "Тип", "Наименование", "Ед.", "Кол-во",
                     "Цена работ", "Цена материалов", "Стоимость работ",
                     "Стоимость материалов", "Итого"])

    # Группировка по разделам
    from itertools import groupby
    items_sorted = sorted(items, key=lambda x: (x.get("section") or "Без раздела"))
    row = 2
    total_work = total_mat = 0.0

    for section, group in groupby(items_sorted, key=lambda x: x.get("section") or "Без раздела"):
        group_list = list(group)
        sec_work = sec_mat = 0.0

        for i, item in enumerate(group_list, 1):
            qty = item.get("quantity", 0)
            wp = item.get("work_price", 0)
            mp = item.get("mat_price", 0)
            cost_w = qty * wp
            cost_m = qty * mp
            sec_work += cost_w
            sec_mat += cost_m

            values = [i, section if i == 1 else "", item.get("type"), item.get("name"),
                      item.get("unit"), qty, wp, mp, cost_w, cost_m, cost_w + cost_m]
            for col, val in enumerate(values, 1):
                cell = ws.cell(row=row, column=col, value=val)
                cell.font = NORMAL_FONT
                cell.border = ALL_BORDERS
                if col in (7, 8, 9, 10, 11):
                    cell.number_format = '#,##0.00'
            row += 1

        # Итог по разделу
        for col in range(1, 12):
            cell = ws.cell(row=row, column=col)
            cell.fill = TOTAL_FILL
            cell.font = Font(bold=True, name="Arial", size=10)
            cell.border = ALL_BORDERS
        ws.cell(row=row, column=3, value=f"Итого по разделу: {section}")
        ws.cell(row=row, column=9, value=sec_work).number_format = '#,##0.00'
        ws.cell(row=row, column=10, value=sec_mat).number_format = '#,##0.00'
        ws.cell(row=row, column=11, value=sec_work + sec_mat).number_format = '#,##0.00'
        total_work += sec_work
        total_mat += sec_mat
        row += 2  # пустая строка между разделами

    # Итоговые строки
    total = total_work + total_mat
    vat_amount = total * vat_rate / 100
    for label, value in [
        ("Итого работы:", total_work),
        ("Итого материалы:", total_mat),
        ("Итого без НДС:", total),
        (f"НДС {vat_rate:.0f}%:", vat_amount),
        ("ИТОГО С НДС:", total + vat_amount),
    ]:
        for col in range(1, 12):
            cell = ws.cell(row=row, column=col)
            cell.fill = TOTAL_FILL
            cell.font = Font(bold=True, name="Arial", size=10)
            cell.border = ALL_BORDERS
        ws.cell(row=row, column=3, value=label)
        ws.cell(row=row, column=11, value=value).number_format = '#,##0.00'
        row += 1

    _auto_width(ws)
    buf = io.BytesIO()
    wb.save(buf)
    return buf.getvalue()
```

---

## PdfService

```python
# app/services/pdf_service.py
"""
Генерация PDF отчёта сравнения (COMPARE_PROJECT_SMETA).
Использует fpdf2 — нет системных зависимостей.
"""
import io
from fpdf import FPDF
from app.config import settings

class SmetaPDF(FPDF):
    def header(self):
        self.set_font("Helvetica", "B", 14)
        self.set_text_color(46, 95, 163)
        self.cell(0, 10, "Smeta AI — Отчёт сравнения", align="C", new_x="LMARGIN", new_y="NEXT")
        self.ln(2)

    def footer(self):
        self.set_y(-15)
        self.set_font("Helvetica", "I", 8)
        self.set_text_color(128)
        self.cell(0, 10, f"Стр. {self.page_no()}", align="C")


def generate_comparison_report(comparison_data: dict) -> bytes:
    """
    comparison_data — распарсенный JSON-ответ Claude для COMPARE_PROJECT_SMETA.
    Структура: {summary, discrepancies, missing_in_smeta, missing_in_project, recommendations}
    """
    pdf = SmetaPDF(orientation="P", unit="mm", format="A4")
    pdf.add_page()
    pdf.set_auto_page_break(auto=True, margin=15)

    summary = comparison_data.get("summary", {})
    discrepancies = comparison_data.get("discrepancies", [])
    recommendations = comparison_data.get("recommendations", [])

    # Сводные карточки
    pdf.set_font("Helvetica", "B", 12)
    pdf.set_text_color(0)
    pdf.cell(0, 8, "Сводная информация", new_x="LMARGIN", new_y="NEXT")
    pdf.set_font("Helvetica", size=10)

    for label, value in [
        ("Итого по проекту:", f"{summary.get('project_total', 0):,.2f} руб."),
        ("Итого по смете:", f"{summary.get('smeta_total', 0):,.2f} руб."),
        ("Расхождение:", f"{summary.get('difference', 0):,.2f} руб. ({summary.get('difference_pct', 0):.1f}%)"),
    ]:
        pdf.cell(70, 7, label)
        pdf.cell(0, 7, value, new_x="LMARGIN", new_y="NEXT")
    pdf.ln(5)

    # Таблица расхождений
    if discrepancies:
        pdf.set_font("Helvetica", "B", 11)
        pdf.cell(0, 8, "Расхождения", new_x="LMARGIN", new_y="NEXT")
        pdf.set_font("Helvetica", "B", 9)

        col_widths = [80, 30, 30, 25, 25]
        headers = ["Наименование", "По проекту", "По смете", "Разница %", "Статус"]
        for w, h in zip(col_widths, headers):
            pdf.cell(w, 7, h, border=1)
        pdf.ln()

        pdf.set_font("Helvetica", size=9)
        for d in discrepancies:
            severity_color = {
                "critical": (192, 57, 43),
                "warning": (214, 137, 16),
                "ok": (30, 126, 52),
            }.get(d.get("severity", "ok"), (0, 0, 0))

            pdf.cell(80, 6, str(d.get("name", ""))[:50], border=1)
            pdf.cell(30, 6, f"{d.get('project_value') or '-'}", border=1, align="R")
            pdf.cell(30, 6, f"{d.get('smeta_value') or '-'}", border=1, align="R")
            pdf.cell(25, 6, f"{d.get('difference_pct', 0):.1f}%", border=1, align="R")
            pdf.set_text_color(*severity_color)
            pdf.cell(25, 6, d.get("severity", ""), border=1, align="C")
            pdf.set_text_color(0)
            pdf.ln()
        pdf.ln(5)

    # Рекомендации
    if recommendations:
        pdf.set_font("Helvetica", "B", 11)
        pdf.cell(0, 8, "Рекомендации", new_x="LMARGIN", new_y="NEXT")
        pdf.set_font("Helvetica", size=10)
        for i, rec in enumerate(recommendations, 1):
            pdf.multi_cell(0, 6, f"{i}. {rec}")
        pdf.ln(3)

    buf = io.BytesIO()
    pdf.output(buf)
    return buf.getvalue()
```

---

## SnapshotService

```python
# app/services/snapshot_service.py
import uuid
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, delete
from app.models.task_version import TaskVersion
from app.models.estimate_item import EstimateItem

class SnapshotService:

    async def save_snapshot(
        self,
        db: AsyncSession,
        task_id: uuid.UUID,
        change_type: str,
        change_description: str | None,
        created_by: str,
    ) -> TaskVersion:
        # Получить текущие позиции
        items = (await db.execute(
            select(EstimateItem).where(EstimateItem.task_id == task_id)
            .order_by(EstimateItem.position)
        )).scalars().all()

        # Сериализовать
        snapshot = {"items": [
            {
                "id": str(item.id),
                "position": item.position,
                "type": item.type,
                "name": item.name,
                "unit": item.unit,
                "quantity": item.quantity,
                "work_price": item.work_price,
                "mat_price": item.mat_price,
                "section": item.section,
                "notes": item.notes,
                "is_analogue": item.is_analogue,
                "original_item_id": str(item.original_item_id) if item.original_item_id else None,
                "analogue_note": item.analogue_note,
                "extra": item.extra,
            }
            for item in items
        ]}

        # Вычислить следующий version_number
        result = await db.execute(
            select(TaskVersion.version_number)
            .where(TaskVersion.task_id == task_id)
            .order_by(TaskVersion.version_number.desc())
            .limit(1)
        )
        last = result.scalar_one_or_none()
        next_number = (last or 0) + 1

        version = TaskVersion(
            task_id=task_id,
            version_number=next_number,
            snapshot=snapshot,
            change_description=change_description,
            change_type=change_type,
            created_by=created_by,
        )
        db.add(version)
        await db.commit()
        await db.refresh(version)
        return version

    async def restore_snapshot(
        self,
        db: AsyncSession,
        task_id: uuid.UUID,
        version_id: uuid.UUID,
    ):
        # Получить версию
        version = await db.get(TaskVersion, version_id)
        if not version or version.task_id != task_id:
            raise ValueError("Version not found")

        # Сохранить текущее состояние перед восстановлением
        await self.save_snapshot(db, task_id, "restoration", f"Перед восстановлением v{version.version_number}", "system")

        # Удалить текущие позиции
        await db.execute(delete(EstimateItem).where(EstimateItem.task_id == task_id))

        # Воссоздать из snapshot с оригинальными UUID
        for item_data in version.snapshot["items"]:
            item = EstimateItem(
                id=uuid.UUID(item_data["id"]),
                task_id=task_id,
                position=item_data["position"],
                type=item_data["type"],
                name=item_data["name"],
                unit=item_data["unit"],
                quantity=item_data["quantity"],
                work_price=item_data["work_price"],
                mat_price=item_data["mat_price"],
                section=item_data.get("section"),
                notes=item_data.get("notes"),
                is_analogue=item_data.get("is_analogue", False),
                original_item_id=uuid.UUID(item_data["original_item_id"]) if item_data.get("original_item_id") else None,
                analogue_note=item_data.get("analogue_note"),
                extra=item_data.get("extra"),
            )
            db.add(item)

        await db.commit()

snapshot_service = SnapshotService()
```

---

## OptimizationService

```python
# app/services/optimization_service.py
import uuid
import json
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.models.estimate_item import EstimateItem
from app.models.task import Task
from app.services.snapshot_service import snapshot_service
from app.services.claude_service import claude_service

class OptimizationService:

    async def get_optimization_plan(self, db: AsyncSession, task_id: uuid.UUID) -> dict:
        """
        Вернуть топ-10 позиций по стоимости для потенциальной оптимизации.
        """
        items = (await db.execute(
            select(EstimateItem).where(EstimateItem.task_id == task_id)
        )).scalars().all()

        # Вычислить стоимость каждой позиции
        items_with_cost = [
            {
                "id": str(item.id),
                "name": item.name,
                "type": item.type,
                "unit": item.unit,
                "quantity": item.quantity,
                "work_price": item.work_price,
                "mat_price": item.mat_price,
                "total_cost": (item.work_price + item.mat_price) * item.quantity,
            }
            for item in items if not item.is_analogue
        ]
        items_with_cost.sort(key=lambda x: x["total_cost"], reverse=True)
        top10 = items_with_cost[:10]

        total_cost = sum(i["total_cost"] for i in items_with_cost)
        top10_cost = sum(i["total_cost"] for i in top10)

        return {
            "items": top10,
            "total_cost": total_cost,
            "top10_cost": top10_cost,
            "top10_pct": (top10_cost / total_cost * 100) if total_cost else 0,
        }

    async def execute_optimization(
        self,
        db: AsyncSession,
        task_id: uuid.UUID,
        item_ids: list[str],
    ):
        """
        Для каждой позиции из item_ids — запросить у Claude более дешёвый аналог.
        Батчи по 10 позиций.
        """
        # Сохранить snapshot перед изменениями
        await snapshot_service.save_snapshot(
            db, task_id, "optimization", "Перед оптимизацией", "system"
        )

        # Загрузить позиции
        items = (await db.execute(
            select(EstimateItem).where(
                EstimateItem.id.in_([uuid.UUID(i) for i in item_ids]),
                EstimateItem.task_id == task_id,
            )
        )).scalars().all()

        # Батчи по 10
        batch_size = 10
        for i in range(0, len(items), batch_size):
            batch = items[i:i + batch_size]
            await self._optimize_batch(db, batch)

        # Обновить estimate_status
        task = await db.get(Task, task_id)
        if task:
            task.estimate_status = "optimized"
            task.estimate_status_updated_by = "auto"
        await db.commit()

    async def _optimize_batch(self, db: AsyncSession, items: list):
        items_json = json.dumps([
            {"id": str(it.id), "name": it.name, "type": it.type,
             "unit": it.unit, "work_price": it.work_price, "mat_price": it.mat_price}
            for it in items
        ], ensure_ascii=False)

        prompt = f"""
Для каждой позиции найди более дешёвый аналог если возможно.
Позиции: {items_json}

Ответь ТОЛЬКО валидным JSON:
{{
  "results": [
    {{
      "id": "uuid",
      "found": true | false,
      "new_work_price": число или null,
      "new_mat_price": число или null,
      "analogue_note": "обоснование"
    }}
  ]
}}
"""
        try:
            response = await claude_service.complete(
                messages=[{"role": "user", "content": prompt}],
                task_type="SMETA_FROM_TZ",
                use_web_search=True,
            )
            import re
            clean = re.sub(r"```(?:json)?\s*", "", response).strip().rstrip("`")
            data = json.loads(clean)

            for result in data.get("results", []):
                if not result.get("found"):
                    continue
                item = next((it for it in items if str(it.id) == result["id"]), None)
                if not item:
                    continue
                if result.get("new_work_price") and result["new_work_price"] < item.work_price:
                    item.work_price = result["new_work_price"]
                    item.is_analogue = True
                    item.analogue_note = result.get("analogue_note", "")
                if result.get("new_mat_price") and result["new_mat_price"] < item.mat_price:
                    item.mat_price = result["new_mat_price"]
                    item.is_analogue = True
                    item.analogue_note = result.get("analogue_note", "")
        except Exception:
            pass  # Батч не удался — пропустить, не падать

optimization_service = OptimizationService()
```

---

## AnalogueService

```python
# app/services/analogue_service.py
import uuid, json, re
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.models.estimate_item import EstimateItem
from app.services.snapshot_service import snapshot_service
from app.services.claude_service import claude_service
from app.config import settings

class AnalogueService:

    async def find_analogues(
        self, db: AsyncSession, task_id: uuid.UUID, item_id: uuid.UUID
    ) -> list[dict]:
        item = await db.get(EstimateItem, item_id)
        if not item:
            raise ValueError("Item not found")

        # Проверить кэш
        if item.extra and item.extra.get("analogues_cache"):
            return item.extra["analogues_cache"]

        prompt = f"""
Найди 3 аналога для: "{item.name}", ед.изм.: {item.unit}, город: {settings.search_city}.
Текущая цена: {item.mat_price or item.work_price} руб.

Ответь ТОЛЬКО валидным JSON:
{{
  "analogues": [
    {{
      "name": "название аналога",
      "price": число,
      "unit": "ед.изм.",
      "supplier": "поставщик",
      "economy_pct": число,
      "source_url": "url или null"
    }}
  ]
}}
"""
        response = await claude_service.complete(
            messages=[{"role": "user", "content": prompt}],
            task_type="SMETA_FROM_TZ",
            use_web_search=True,
        )
        clean = re.sub(r"```(?:json)?\s*", "", response).strip().rstrip("`")
        data = json.loads(clean)
        analogues = data.get("analogues", [])

        # Кэшировать в extra
        item.extra = {**(item.extra or {}), "analogues_cache": analogues}
        await db.commit()
        return analogues

    async def apply_analogue(
        self,
        db: AsyncSession,
        task_id: uuid.UUID,
        item_id: uuid.UUID,
        analogue_data: dict,
    ):
        await snapshot_service.save_snapshot(
            db, task_id, "manual_edit", f"Применён аналог: {analogue_data.get('name')}", "user"
        )

        original = await db.get(EstimateItem, item_id)
        if not original:
            raise ValueError("Item not found")

        # Получить max position
        result = await db.execute(
            select(EstimateItem.position)
            .where(EstimateItem.task_id == task_id)
            .order_by(EstimateItem.position.desc()).limit(1)
        )
        max_pos = result.scalar_one_or_none() or 0

        new_item = EstimateItem(
            task_id=task_id,
            position=original.position,  # Вставить на место оригинала
            type=original.type,
            name=analogue_data["name"],
            unit=analogue_data.get("unit", original.unit),
            quantity=original.quantity,
            work_price=analogue_data["price"] if original.type == "Работа" else 0.0,
            mat_price=analogue_data["price"] if original.type == "Материал" else 0.0,
            section=original.section,
            is_analogue=True,
            original_item_id=original.id,
            analogue_note=f"Аналог от {analogue_data.get('supplier', '')}. {analogue_data.get('source_url', '')}",
        )
        db.add(new_item)
        # Скрыть оригинал: сдвинуть его в конец
        original.position = max_pos + 1
        await db.commit()

    async def revert_analogue(
        self,
        db: AsyncSession,
        task_id: uuid.UUID,
        item_id: uuid.UUID,
    ):
        item = await db.get(EstimateItem, item_id)
        if not item or not item.is_analogue:
            raise ValueError("Not an analogue item")

        await snapshot_service.save_snapshot(
            db, task_id, "manual_edit", "Отмена аналога", "user"
        )

        original_id = item.original_item_id
        await db.delete(item)

        # Вернуть оригинал на его позицию
        if original_id:
            original = await db.get(EstimateItem, original_id)
            if original:
                # Найти позицию аналога и восстановить
                original.position = item.position

        await db.commit()

analogue_service = AnalogueService()
```

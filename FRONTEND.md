# FRONTEND.md — Фронтенд

## Стек

- React 18 + TypeScript (strict: true)
- Vite 5.4
- Zustand 5.0 (state)
- Axios 1.7.9 (HTTP)
- React Router 6

---

## package.json (зависимости)

```json
{
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "react-router-dom": "^6.28.0",
    "axios": "^1.7.9",
    "zustand": "^5.0.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.0.0",
    "typescript": "^5.6.0",
    "vite": "^5.4.0"
  }
}
```

---

## src/api/client.ts

```typescript
import axios from 'axios';
import { useAuthStore } from '../store/auth';

const client = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000',
});

// Добавить JWT к каждому запросу
client.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 401 → разлогинить
client.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default client;
```

---

## src/store/auth.ts

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface AuthState {
  token: string | null;
  role: 'user' | 'admin' | null;
  login: (token: string, role: string) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      token: null,
      role: null,
      login: (token, role) => set({ token, role: role as 'user' | 'admin' }),
      logout: () => set({ token: null, role: null }),
    }),
    { name: 'auth' }
  )
);
```

---

## src/App.tsx

```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import ProtectedRoute from './components/ProtectedRoute';
import Login from './pages/Login';
import TaskCreate from './pages/TaskCreate';
import TaskStatus from './pages/TaskStatus';
import EstimateView from './pages/EstimateView';
import Admin from './pages/Admin';
import Layout from './components/Layout';

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route element={<ProtectedRoute />}>
          <Route element={<Layout />}>
            <Route path="/" element={<Navigate to="/task/create" replace />} />
            <Route path="/task/create" element={<TaskCreate />} />
            <Route path="/task/:id/status" element={<TaskStatus />} />
            <Route path="/task/:id/estimate" element={<EstimateView />} />
          </Route>
        </Route>
        <Route element={<ProtectedRoute adminOnly />}>
          <Route path="/admin" element={<Admin />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

---

## src/components/ProtectedRoute.tsx

```tsx
import { Navigate, Outlet } from 'react-router-dom';
import { useAuthStore } from '../store/auth';

interface Props {
  adminOnly?: boolean;
}

export default function ProtectedRoute({ adminOnly = false }: Props) {
  const { token, role } = useAuthStore();

  if (!token) return <Navigate to="/login" replace />;
  if (adminOnly && role !== 'admin') return <Navigate to="/task/create" replace />;

  return <Outlet />;
}
```

---

## src/pages/Login.tsx

```tsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import client from '../api/client';
import { useAuthStore } from '../store/auth';

export default function Login() {
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  const navigate = useNavigate();
  const login = useAuthStore((s) => s.login);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError('');
    try {
      const { data } = await client.post('/auth/login', { password });
      login(data.access_token, data.role);
      navigate(data.role === 'admin' ? '/admin' : '/task/create');
    } catch {
      setError('Неверный пароль');
    } finally {
      setLoading(false);
    }
  }

  return (
    <div style={{ display: 'flex', justifyContent: 'center', alignItems: 'center', height: '100vh' }}>
      <form onSubmit={handleSubmit} style={{ width: 320, display: 'flex', flexDirection: 'column', gap: 12 }}>
        <h2>Smeta AI</h2>
        <input
          type="password"
          placeholder="Пароль"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          disabled={loading}
        />
        {error && <p style={{ color: 'red' }}>{error}</p>}
        <button type="submit" disabled={loading}>
          {loading ? 'Вход...' : 'Войти'}
        </button>
      </form>
    </div>
  );
}
```

---

## src/pages/TaskCreate.tsx

Ключевые части:

```tsx
const TASK_TYPES = [
  { value: 'LIST_FROM_TZ', label: 'Перечень из ТЗ' },
  { value: 'LIST_FROM_TZ_PROJECT', label: 'Перечень из ТЗ + проект' },
  { value: 'LIST_FROM_PROJECT', label: 'Перечень из проекта' },
  { value: 'RESEARCH_PROJECT', label: 'Исследование проекта' },
  { value: 'SMETA_FROM_LIST', label: 'Смета из перечня' },
  { value: 'SMETA_FROM_TZ', label: 'Смета из ТЗ' },
  { value: 'SMETA_FROM_TZ_PROJECT', label: 'Смета из ТЗ + проект' },
  { value: 'SMETA_FROM_PROJECT', label: 'Смета из проекта' },
  { value: 'SMETA_FROM_EDC_PROJECT', label: 'Смета из EDC + проект' },
  { value: 'SMETA_FROM_GRAND_PROJECT', label: 'Смета из Grand CAD + проект' },
  { value: 'SCAN_TO_EXCEL', label: 'Скан → Excel' },
  { value: 'COMPARE_PROJECT_SMETA', label: 'Сравнение проекта со сметой' },
];

async function handleSubmit() {
  const formData = new FormData();
  formData.append('task_type', taskType);
  if (prompt) formData.append('prompt', prompt);
  files.forEach((file) => formData.append('files', file));

  const { data } = await client.post('/tasks', formData);
  navigate(`/task/${data.task_id}/status`);
}
```

FileUpload: drag-and-drop зона. При drop/select — проверить count ≤ 5, size ≤ 50 МБ.
Показывать список файлов с кнопкой удаления каждого.

---

## src/pages/TaskStatus.tsx

Логика polling:

```tsx
useEffect(() => {
  const poll = async () => {
    const { data } = await client.get(`/tasks/${id}/status`);
    setTask(data);

    if (['completed', 'failed', 'cancelled'].includes(data.status)) {
      clearInterval(intervalRef.current);
      if (data.status === 'completed') {
        // Загрузить список файлов результата
        const { data: results } = await client.get(`/tasks/${id}/results`);
        setResults(results);
      }
    }
  };

  poll();
  const interval = task?.status === 'processing' ? 2000 : 5000;
  intervalRef.current = setInterval(poll, interval);
  return () => clearInterval(intervalRef.current);
}, [id, task?.status]);
```

Чат-интерфейс:
```tsx
async function sendMessage() {
  await client.post(`/tasks/${id}/message`, { content: chatInput });
  setChatInput('');
  // Polling перезапустится сам при следующем обновлении статуса
}
```

Кнопки при completed:
- Скачать каждый файл из results: `window.open(${VITE_API_BASE_URL}/results/${file.id}/download)`
- "Открыть смету" → navigate(`/task/${id}/estimate`) — только если estimate_status != null

---

## src/pages/EstimateView.tsx

```tsx
// Загрузка данных
const { data } = await client.get(`/projects/estimates/${taskId}/items`);
// data: { items, vat_rate, total_work, total_mat, total, total_vat }

// Отображение таблицы позиций
// Столбцы: Позиция | Раздел | Тип | Наименование | Ед. | Кол-во | Цена работ | Цена мат. | Стоимость | Аналог?

// Итоги внизу: Работы | Материалы | Итого без НДС | НДС X% | ИТОГО
```

Компоненты на странице:
- `StatusBadge` с estimate_status
- Кнопка "История версий" → открывает `VersionHistoryDrawer`
- Кнопка "Оптимизировать" → открывает `OptimizationChecklist`
- На каждой строке с type='Материал': кнопка "Аналоги" → открывает `AnaloguePanel` для этой позиции

---

## src/components/StatusBadge.tsx

```tsx
const STATUS_CONFIG = {
  pending:    { label: 'Ожидание',     color: '#888' },
  processing: { label: 'Обработка...', color: '#2196f3', animated: true },
  completed:  { label: 'Завершено',    color: '#4caf50' },
  failed:     { label: 'Ошибка',       color: '#f44336' },
  cancelled:  { label: 'Отменено',     color: '#ff9800' },
  // estimate statuses
  uploaded:   { label: 'Загружено',    color: '#9e9e9e' },
  calculated: { label: 'Рассчитано',   color: '#2196f3' },
  optimized:  { label: 'Оптимизировано', color: '#4caf50' },
};
```

---

## src/components/Layout.tsx

```tsx
// Содержит:
// - Хедер: логотип, имя пользователя, кнопка выхода
// - Слева: ProjectsSidebar (коллапсируемый)
// - Центр: <Outlet /> (контент страницы)
```

---

## src/components/ProjectsSidebar.tsx

```tsx
// Загрузить список проектов: GET /projects
// Кнопка "Новый проект" → POST /projects
// Клик на проект → GET /projects/{id} → показать задачи в проекте
// Drag задачи в проект → POST /projects/{pid}/estimates/{tid}
```

---

## src/components/VersionHistoryDrawer.tsx

```tsx
// GET /projects/estimates/{taskId}/versions
// Список: version_number, created_at, change_type, change_description
// Кнопка "Восстановить" → POST /projects/estimates/{taskId}/versions/{vid}/restore
// Confirm dialog перед восстановлением
```

---

## src/components/OptimizationChecklist.tsx

```tsx
// 1. При открытии: POST /projects/estimates/{taskId}/optimize/plan
//    → получить список позиций + ожидаемую экономию
// 2. Показать чеклист с возможностью снять/поставить галочки
// 3. Кнопка "Запустить оптимизацию"
// 4. POST /projects/estimates/{taskId}/optimize/execute с выбранными item_ids
// 5. После успеха: перезагрузить позиции сметы
```

---

## src/components/AnaloguePanel.tsx

```tsx
// 1. При открытии: POST /projects/estimates/{taskId}/items/{itemId}/find-analogues
//    → список аналогов [{name, price, unit, supplier, economy_pct, source_url}]
// 2. Для каждого аналога: карточка с ценой, поставщиком, % экономии, ссылкой
// 3. Кнопка "Применить" → POST .../apply-analogue
// 4. Если уже аналог (is_analogue=true): кнопка "Отменить" → POST .../revert-analogue
```

---

## src/pages/Admin.tsx

```tsx
// Вкладка 1: Задачи
// - Фильтры: status, task_type, date_from, date_to
// - GET /admin/tasks?page=1&page_size=20&status=...
// - Таблица: id, task_type, status, created_at, действия
// - Действия: просмотр деталей, скачать входной файл, удалить

// Вкладка 2: Прайс-листы
// - GET /admin/price-lists/info → показать имена файлов и дату обновления
// - Кнопка загрузить прайс работ → POST /admin/price-lists/works
// - Кнопка загрузить прайс материалов → POST /admin/price-lists/materials
```

---

## vite.config.ts

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
});
```

---

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "jsx": "react-jsx",
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

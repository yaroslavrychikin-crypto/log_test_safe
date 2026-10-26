# 📘 Часть 9: Context API и глобальное состояние

## 9.1 Проблема prop drilling

До сих пор данные передавались через пропсы сверху вниз. Но представь: `App` хранит `theme`, а использовать её нужно в `Button`, который лежит на 5 уровней глубже.

```jsx
App (theme) → Layout (theme) → Sidebar (theme) → Menu (theme) → Button (theme)
```

Каждый промежуточный компонент получает `theme`, хотя **сам её не использует** — просто передаёт дальше. Это **prop drilling**.

**Проблемы:**
- Много «мусорных» пропсов в компонентах
- При добавлении нового уровня — переписывать всю цепочку
- Рефакторинг становится болью

**Решение:** Context API — способ передать значение **сразу всем потомкам**, минуя промежуточные компоненты.

---

## 9.2 Context API: три шага

### Шаг 1: Создать контекст

```jsx
import { createContext } from 'react';

// Аргумент — значение по умолчанию (используется, если нет Provider выше)
export const ThemeContext = createContext(null);
```

**Важно:** контекст — это не «хранилище». Это **канал**, по которому значение передаётся от Provider к потребителям.

### Шаг 2: Обернуть дерево в Provider

```jsx
import { ThemeContext } from './ThemeContext';
import { useState } from 'react';

function App() {
  const [theme, setTheme] = useState('light');

  return (
    // value — то, что получат все потомки
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext.Provider>
  );
}
```

**Ключевой момент:** `value` — это **объект**. Если передать `{ theme, setTheme }` inline, при каждом рендере `App` создаётся **новый объект** → все потребители перерисовываются. Об этом поговорим в Дне 10 (и это — главная ловушка Context).

### Шаг 3: Использовать через `useContext`

```jsx
import { useContext } from 'react';
import { ThemeContext } from './ThemeContext';

function Button() {
  const { theme, setTheme } = useContext(ThemeContext);

  return (
    <button
      onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}
      className={theme}
    >
      Переключить тему
    </button>
  );
}
```

`Button` может лежать на **любой** глубине — главное, чтобы он был **внутри** `ThemeContext.Provider`.

---

## 9.3 Паттерн «два контекста» для состояния и dispatch

Когда контекст содержит и данные, и функцию их изменения, **любое** изменение данных перерисовывает **всех** потребителей — даже тех, кому нужна только функция.

**Решение от команды React:** разделить на **два контекста** :

```jsx
// context/TasksContext.js
import { createContext } from 'react';

// Контекст только для данных
export const TasksContext = createContext(null);

// Контекст только для функции изменения
export const TasksDispatchContext = createContext(null);
```

```jsx
// App.jsx
import { useReducer } from 'react';
import { TasksContext, TasksDispatchContext } from './context/TasksContext';

function App() {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);

  return (
    <TasksContext.Provider value={tasks}>
      <TasksDispatchContext.Provider value={dispatch}>
        <Layout />
      </TasksDispatchContext.Provider>
    </TasksContext.Provider>
  );
}
```

**Что даёт:**
- Компонент, который только **отображает** задачи → подписан на `TasksContext` → перерисовывается при изменении `tasks`
- Компонент, который только **меняет** задачи (например, форма добавления) → подписан на `TasksDispatchContext` → `dispatch` **никогда не меняется** → **не перерисовывается** при изменении списка

Это стандартный паттерн из официальной документации React .

---

## 9.4 Кастомный хук для контекста

Вместо `useContext(TasksContext)` в каждом компоненте — обернём в хук с проверкой:

```jsx
// hooks/useTasks.js
import { useContext } from 'react';
import { TasksContext, TasksDispatchContext } from '../context/TasksContext';

export function useTasks() {
  const context = useContext(TasksContext);
  if (context === null) {
    throw new Error('useTasks must be used within TasksProvider');
  }
  return context;
}

export function useTasksDispatch() {
  const context = useContext(TasksDispatchContext);
  if (context === null) {
    throw new Error('useTasksDispatch must be used within TasksProvider');
  }
  return context;
}
```

**Зачем:**
1. Понятная ошибка, если компонент забыли обернуть в Provider (а не `undefined is not a function`)
2. Один импорт вместо двух
3. Легко добавить логику (например, логгирование) в одном месте

---

## 9.5 Когда Context — **не** решение

Context — не замена Redux/Zustand для всего. Используй его, когда:
- Данные нужны **многим** компонентам на **разных** уровнях
- Данные меняются **не часто** (тема, локаль, авторизация)
- Это **не** высокочастотные обновления (не для курсора мыши, скролла)

**Для сложного состояния с частыми обновлениями** лучше Zustand/Redux — они оптимизируют подписки через селекторы.

---

## 9.6 🎯 Практика Части  9: Todo с Context (рефакторинг Дня 8)

Возьмём todo из Части 8 и уберём prop drilling.

### `src/context/TasksContext.jsx`

```jsx
import { createContext, useReducer, useEffect } from 'react';

// ===== Reducer (логика изменения состояния) =====
const ACTIONS = {
  ADD: 'ADD',
  TOGGLE: 'TOGGLE',
  DELETE: 'DELETE',
  HYDRATE: 'HYDRATE',
};

function tasksReducer(state, action) {
  switch (action.type) {
    case ACTIONS.ADD:
      // Новые задачи сверху
      return [action.payload, ...state];

    case ACTIONS.TOGGLE:
      // Иммутабельное обновление
      return state.map((t) =>
        t.id === action.payload ? { ...t, done: !t.done } : t
      );

    case ACTIONS.DELETE:
      return state.filter((t) => t.id !== action.payload);

    case ACTIONS.HYDRATE:
      // Восстановление из localStorage
      return action.payload;

    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

// ===== Два контекста =====
export const TasksContext = createContext(null);
export const TasksDispatchContext = createContext(null);

// ===== Провайдер =====
const STORAGE_KEY = 'tasks-context:v1';

export function TasksProvider({ children }) {
  // Ленивая инициализация из localStorage
  const [tasks, dispatch] = useReducer(tasksReducer, [], () => {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      return saved ? JSON.parse(saved) : [];
    } catch {
      return [];
    }
  });

  // Сохранение при каждом изменении
  useEffect(() => {
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(tasks));
    } catch (err) {
      console.warn('Не удалось сохранить задачи', err);
    }
  }, [tasks]);

  return (
    <TasksContext.Provider value={tasks}>
      <TasksDispatchContext.Provider value={dispatch}>
        {children}
      </TasksDispatchContext.Provider>
    </TasksContext.Provider>
  );
}
```

### `src/hooks/useTasks.js`

```jsx
import { useContext } from 'react';
import { TasksContext, TasksDispatchContext } from '../context/TasksContext';

export function useTasks() {
  const ctx = useContext(TasksContext);
  if (ctx === null) {
    throw new Error('useTasks must be used within TasksProvider');
  }
  return ctx;
}

export function useTasksDispatch() {
  const ctx = useContext(TasksDispatchContext);
  if (ctx === null) {
    throw new Error('useTasksDispatch must be used within TasksProvider');
  }
  return ctx;
}
```

### `src/components/AddTask.jsx` — только dispatch, без данных

```jsx
import { useState } from 'react';
import { useTasksDispatch } from '../hooks/useTasks';

function AddTask() {
  const dispatch = useTasksDispatch();
  const [text, setText] = useState('');

  function handleSubmit(e) {
    e.preventDefault();
    const trimmed = text.trim();
    if (!trimmed) return;

    dispatch({
      type: 'ADD',
      payload: {
        id: crypto.randomUUID(),
        text: trimmed,
        done: false,
      },
    });
    setText('');
  }

  return (
    <form onSubmit={handleSubmit} style={{ display: 'flex', gap: 8 }}>
      <input
        value={text}
        onChange={(e) => setText(e.target.value)}
        placeholder="Новая задача"
        style={{ flex: 1, padding: 8 }}
      />
      <button type="submit">Добавить</button>
    </form>
  );
}

// ⚠️ Этот компонент НЕ подписан на TasksContext — только на Dispatch.
// При добавлении задачи он НЕ перерисуется (dispatch стабилен).
export default AddTask;
```

### `src/components/TaskList.jsx` — только данные

```jsx
import { useTasks, useTasksDispatch } from '../hooks/useTasks';

function TaskList() {
  const tasks = useTasks(); // подписан на данные
  const dispatch = useTasksDispatch(); // подписан на dispatch (стабилен)

  if (tasks.length === 0) {
    return <p style={{ color: '#999' }}>Задач нет</p>;
  }

  return (
    <ul style={{ listStyle: 'none', padding: 0 }}>
      {tasks.map((task) => (
        <li key={task.id} style={{ display: 'flex', gap: 8, padding: '8px 0' }}>
          <input
            type="checkbox"
            checked={task.done}
            onChange={() => dispatch({ type: 'TOGGLE', payload: task.id })}
          />
          <span style={{ flex: 1, textDecoration: task.done ? 'line-through' : 'none' }}>
            {task.text}
          </span>
          <button onClick={() => dispatch({ type: 'DELETE', payload: task.id })}>
            ✕
          </button>
        </li>
      ))}
    </ul>
  );
}

export default TaskList;
```

### `src/App.jsx`

```jsx
import { TasksProvider } from './context/TasksContext';
import AddTask from './components/AddTask';
import TaskList from './components/TaskList';

function App() {
  return (
    <TasksProvider>
      <div style={{ maxWidth: 500, margin: '40px auto', padding: 20 }}>
        <h1>Todo с Context</h1>
        <AddTask />
        <TaskList />
      </div>
    </TasksProvider>
  );
}

export default App;
```

**Что изменилось:**
- `App` больше не хранит `tasks` — это делает `TasksProvider`
- `AddTask` и `TaskList` не получают пропсов — берут из контекста
- `AddTask` **не перерисовывается** при изменении списка (подписан только на dispatch)

**✅ Чек-лист Части 9:**
- [ ] Понимаю, что такое prop drilling и почему это проблема
- [ ] Умею создавать контекст, Provider и использовать `useContext`
- [ ] Знаю паттерн «два контекста» (данные + dispatch)
- [ ] Пишу кастомные хуки с проверкой на null
- [ ] Понимаю, когда Context — правильный выбор, а когда — нет

---

# 📘 Часть 10: Продвинутые хуки

## 10.1 `useReducer` — когда `useState` не хватает

`useState` хорош для простых значений. Когда логика обновления сложная (много вариантов), `useReducer` чище.

### Синтаксис

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

- **`reducer(state, action)`** — чистая функция: получает текущее состояние и действие, возвращает **новое** состояние
- **`dispatch(action)`** — отправляет действие в reducer

### Пример: счётчик с историей

```jsx
function counterReducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: 0 };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });

  return (
    <div>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+1</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-1</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Сброс</button>
    </div>
  );
}
```

### `useState` vs `useReducer`

| | `useState` | `useReducer` |
|---|---|---|
| Для чего | Простые значения | Сложная логика, много действий |
| Обновление | Прямо в компоненте | В отдельной функции-reducer |
| Тестирование | Сложнее | Reducer — чистая функция, легко тестировать |
| Масштабирование | Много `setX` в компоненте | Вся логика в одном месте |

**Правило:** если у тебя больше 3–4 вариантов обновления одного состояния — переходи на `useReducer`.

---

## 10.2 `React.memo` — мемоизация компонента

`React.memo` — HOC (higher-order component), который **не даёт компоненту перерисоваться**, если его пропсы не изменились (поверхностное сравнение).

```jsx
import { memo } from 'react';

const ExpensiveComponent = memo(function ExpensiveComponent({ value }) {
  console.log('render');
  return <div>{value}</div>;
});
```

**Когда работает:**
- Родитель перерисовался, но пропсы `ExpensiveComponent` **не изменились** (по `===`) → рендер пропускается

**Когда НЕ работает:**
- Пропсы — **новые объекты/функции** при каждом рендере родителя (даже с теми же данными) → `===` даёт `false` → рендер происходит

**Решение:** стабилизировать пропсы через `useMemo` (для значений) и `useCallback` (для функций).

---

## 10.3 `useMemo` — мемоизация значения

`useMemo` кэширует **результат вычисления** и пересчитывает только при изменении зависимостей .

```jsx
const memoizedValue = useMemo(() => expensiveCalculation(a, b), [a, b]);
```

### Когда использовать

1. **Дорогие вычисления** — фильтрация/сортировка большого массива, сложные расчёты
2. **Стабильные объекты/массивы** как пропсы для `memo`-компонентов

```jsx
// Без useMemo: новый объект при каждом рендере → memo-ребёнок перерисовывается
const config = { userId, theme };

// С useMemo: ссылка та же, пока зависимости не изменились
const config = useMemo(() => ({ userId, theme }), [userId, theme]);
```

### Когда НЕ использовать

```jsx
// ❌ Бессмысленно: сложение быстрее, чем накладные расходы useMemo
const sum = useMemo(() => a + b, [a, b]);

// ✅ Просто считай
const sum = a + b;
```

**Правило:** `useMemo` имеет смысл только для **реально дорогих** вычислений или для **стабилизации ссылок** перед `memo` .

---

## 10.4 `useCallback` — мемоизация функции

`useCallback` возвращает **ту же самую функцию**, пока зависимости не изменились .

```jsx
const handleClick = useCallback((id) => {
  console.log(id);
}, []); // пустые зависимости — функция никогда не меняется
```

### Зачем это нужно

Функции в JavaScript — **ссылочные типы**. При каждом рендере создаётся **новая функция**:

```jsx
// При каждом рендере Parent — новый объект функции
<Child onClick={() => console.log('click')} />
```

Если `Child` обёрнут в `memo`, он всё равно перерисуется, потому что `onClick` — **новая ссылка**.

```jsx
// ✅ Стабильная ссылка — Child не перерисовывается
const handleClick = useCallback(() => console.log('click'), []);
<Child onClick={handleClick} />
```

### Когда использовать

1. Функция передаётся в `memo`-компонент
2. Функция — зависимость `useEffect` (иначе эффект может зациклиться) 

### Когда НЕ использовать

```jsx
// ❌ Бессмысленно: функция не передаётся в memo-компонент и не в зависимостях
const handleChange = useCallback((e) => setValue(e.target.value), []);
```

---

## 10.5 «Каскад мемоизации»

Три инструмента работают вместе :

```
Родитель рендерится
  ↓
useMemo → стабильный объект
useCallback → стабильная функция
  ↓
React.memo-ребёнок получает ТЕ ЖЕ пропсы
  ↓
Поверхностное сравнение → пропуск рендера
```

**Без всех трёх — цепочка рвётся:**
- Без `useMemo` → новый объект → `memo` не сработает
- Без `useCallback` → новая функция → `memo` не сработает
- Без `memo` → ребёнок перерисуется в любом случае

---

## 10.6 Кастомные хуки

**Кастомный хук** — функция, начинающаяся с `use`, которая использует другие хуки. Позволяет вынести повторяющуюся логику.

### Пример: `useDebounce`

```jsx
import { useEffect, useState } from 'react';

export function useDebounce(value, delay = 500) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer); // cleanup: отменяем при новом значении
  }, [value, delay]);

  return debounced;
}

// Использование: запрос уходит только через 500 мс после последнего ввода
function Search() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);

  useEffect(() => {
    if (!debouncedQuery) return;
    fetch(`/api/search?q=${debouncedQuery}`);
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

### Пример: `useLocalStorage` (из Части 7)

Уже написан — это тоже кастомный хук.

**Правила:**
- Имя начинается с `use`
- Внутри можно вызывать другие хуки
- Возвращает то, что нужно (значение, массив, объект)
- Не вызывает побочных эффектов вне `useEffect`

---

## 10.7 🎯 Практика Части 10: Оптимизация Todo с Context

Доработаем todo из Части 9: добавим `memo`, `useCallback`, `useMemo` и покажем, как **не** перерисовывать лишнее.

### `src/components/TaskItem.jsx` — memo-компонент

```jsx
import { memo, useCallback } from 'react';
import { useTasksDispatch } from '../hooks/useTasks';

// memo: не перерисовывается, если task и dispatch не изменились
const TaskItem = memo(function TaskItem({ task }) {
  const dispatch = useTasksDispatch();

  // useCallback: стабильные обработчики
  const handleToggle = useCallback(() => {
    dispatch({ type: 'TOGGLE', payload: task.id });
  }, [dispatch, task.id]);

  const handleDelete = useCallback(() => {
    dispatch({ type: 'DELETE', payload: task.id });
  }, [dispatch, task.id]);

  console.log('render TaskItem', task.text);

  return (
    <li style={{ display: 'flex', gap: 8, padding: '8px 0' }}>
      <input type="checkbox" checked={task.done} onChange={handleToggle} />
      <span style={{ flex: 1, textDecoration: task.done ? 'line-through' : 'none' }}>
        {task.text}
      </span>
      <button onClick={handleDelete}>✕</button>
    </li>
  );
});

export default TaskItem;
```

### `src/components/TaskList.jsx` — useMemo для фильтрации

```jsx
import { useMemo, useState } from 'react';
import { useTasks } from '../hooks/useTasks';
import TaskItem from './TaskItem';

function TaskList() {
  const tasks = useTasks();
  const [filter, setFilter] = useState('all');

  // useMemo: фильтрация пересчитывается ТОЛЬКО при изменении tasks или filter
  const visibleTasks = useMemo(() => {
    console.log('пересчёт фильтрации');
    if (filter === 'active') return tasks.filter((t) => !t.done);
    if (filter === 'done') return tasks.filter((t) => t.done);
    return tasks;
  }, [tasks, filter]);

  return (
    <div>
      <div style={{ display: 'flex', gap: 8, marginBottom: 12 }}>
        {['all', 'active', 'done'].map((f) => (
          <button
            key={f}
            onClick={() => setFilter(f)}
            style={{ fontWeight: filter === f ? 'bold' : 'normal' }}
          >
            {f}
          </button>
        ))}
      </div>

      {visibleTasks.length === 0 ? (
        <p>Пусто</p>
      ) : (
        <ul style={{ listStyle: 'none', padding: 0 }}>
          {visibleTasks.map((task) => (
            // TaskItem — memo. task — объект из массива.
            // При toggle меняется ТОЛЬКО один объект → остальные не перерисовываются
            <TaskItem key={task.id} task={task} />
          ))}
        </ul>
      )}
    </div>
  );
}

export default TaskList;
```

**Что происходит при `TOGGLE`:**
1. Reducer создаёт новый массив, но **объекты остальных задач — те же ссылки**
2. `TaskList` перерисовывается (подписан на `TasksContext`)
3. `useMemo` видит, что `tasks` изменился → пересчитывает `visibleTasks`
4. `TaskItem` для изменённой задачи получает **новый** `task` → перерисовывается
5. `TaskItem` для **остальных** получают **те же** `task` → `memo` пропускает рендер

**Проверь в консоли:** при toggle должен быть **один** `render TaskItem`, а не все.

---

## 🏋️ Упражнения для закрепления

### Часть 9
1. **Тема через Context**: переключатель light/dark, значение в `localStorage`.
2. **Авторизация через Context**: `user`, `login()`, `logout()`, проверка в защищённом маршруте.
3. **Корзина через Context + useReducer**: товары, количество, итог.

### Часть 10
4. **`useDebounce`**: поиск с задержкой, покажи «печатает...» пока debounce не сработал.
5. **`usePrevious`**: кастомный хук, возвращающий предыдущее значение.
6. **Оптимизация списка**: оберни элементы в `memo`, проверь через `console.log`, что лишние не рендерятся.

---

## 📋 Чек-лист самопроверки

**Часть 9:**
- [ ] Понимаю, что такое prop drilling и когда Context его решает
- [ ] Умею создавать контекст, Provider и использовать `useContext`
- [ ] Знаю паттерн «два контекста» (данные + dispatch)
- [ ] Пишу кастомные хуки для контекста с проверкой на null
- [ ] Понимаю, что Context — не замена Redux для всего

**Часть 10:**
- [ ] Знаю, когда `useReducer` лучше `useState`
- [ ] Понимаю, что `React.memo` сравнивает пропсы поверхностно
- [ ] Знаю, зачем `useMemo` и `useCallback` (стабилизация ссылок)
- [ ] Понимаю «каскад мемоизации»: memo + useMemo + useCallback
- [ ] Не оптимизирую преждевременно (не оборачиваю всё подряд)
- [ ] Умею писать кастомные хуки

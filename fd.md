

# 📘 Часть 5: `useEffect` и работа с API

## 5.1 Зачем нужен `useEffect`

React-компонент — это функция, которая вызывается **при каждом рендере**. Внутри неё нельзя делать «побочные эффекты» напрямую, потому что:

1. Рендер должен быть **чистым** (без побочек)
2. Рендер может вызываться много раз (StrictMode вызывает дважды в dev)
3. Побочки в рендере = непредсказуемое поведение

**Побочные эффекты** — это всё, что взаимодействует с внешним миром:
- Запросы к серверу (`fetch`)
- Работа с `localStorage`, `document`, `window`
- Таймеры (`setTimeout`, `setInterval`)
- Подписки на события
- Ручное изменение DOM (редко нужно)

Для них есть **`useEffect`** — хук, который запускает код **после рендера**.

### Синтаксис

```jsx
import { useEffect } from 'react';

useEffect(() => {
  // побочный эффект
  return () => {
    // cleanup (необязательно) — что сделать перед следующим запуском или размонтированием
  };
}, [dependencies]); // массив зависимостей
```

---

## 5.2 Массив зависимостей — сердце `useEffect`

Поведение эффекта полностью определяется вторым аргументом.

### Случай 1: `[]` — пустой массив

```jsx
useEffect(() => {
  console.log('Смонтировался');
}, []);
```

Эффект выполнится **один раз** после первого рендера. Аналог `componentDidMount`.

⚠️ В **StrictMode** в dev-режиме выполнится **дважды** — это намеренно, чтобы поймать баги с cleanup. В продакшене — один раз.

### Случай 2: без массива

```jsx
useEffect(() => {
  console.log('После каждого рендера');
});
```

Выполняется **после каждого рендера**. Нужно редко — легко получить бесконечный цикл.

### Случай 3: `[dep1, dep2]` — с зависимостями

```jsx
useEffect(() => {
  console.log('userId изменился:', userId);
}, [userId]);
```

Эффект выполнится:
- После первого рендера
- Каждый раз, когда `userId` изменился

**Правило:** все значения из компонента, которые использует эффект, **должны** быть в зависимостях. Это правило ESLint (`react-hooks/exhaustive-deps`) — не игнорируй его.

### Cleanup-функция

```jsx
useEffect(() => {
  const timer = setInterval(() => {
    console.log('тик');
  }, 1000);

  // Возвращаемая функция вызывается:
  // 1. Перед следующим запуском эффекта (если зависимости изменились)
  // 2. При размонтировании компонента
  return () => {
    clearInterval(timer);
  };
}, []);
```

**Зачем cleanup:**
- Отписаться от подписок (WebSocket, EventEmitter)
- Отменить fetch (через AbortController)
- Очистить таймеры
- Убрать слушатели событий

**Без cleanup** — утечки памяти и баги «обновление размонтированного компонента».

### Жизненный цикл эффекта

```
Mount (первый рендер)
  → эффект запускается
Update (зависимости изменились)
  → cleanup от предыдущего запуска
  → эффект запускается снова
Unmount (компонент удалён)
  → cleanup
```

---

## 5.3 Запросы к API через `fetch`

### Базовый шаблон

```jsx
import { useEffect, useState } from 'react';

function UserList() {
  const [users, setUsers] = useState([]);       // данные
  const [loading, setLoading] = useState(true); // загрузка
  const [error, setError] = useState(null);     // ошибка

  useEffect(() => {
    // ❌ Нельзя делать callback useEffect async напрямую
    // useEffect(async () => {...}) — ошибка!
    // Потому что useEffect ожидает функцию, которая вернёт cleanup,
    // а async-функция возвращает Promise

    // ✅ Решение: объявить async-функцию внутри и сразу вызвать
    async function loadUsers() {
      try {
        setLoading(true);
        setError(null);

        const res = await fetch('https://jsonplaceholder.typicode.com/users');

        // ВАЖНО: fetch не бросает ошибку на 404/500 — нужно проверить res.ok
        if (!res.ok) {
          throw new Error(`HTTP ${res.status}`);
        }

        const data = await res.json();
        setUsers(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }

    loadUsers();
  }, []); // пустой массив — загружаем один раз при монтировании

  if (loading) return <p>Загрузка...</p>;
  if (error) return <p style={{ color: 'red' }}>Ошибка: {error}</p>;
  if (users.length === 0) return <p>Список пуст</p>;

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Три состояния загрузки

Всегда держи **три состояния** для любого запроса:

| Состояние | Значение |
|-----------|----------|
| `loading` | `true` пока идёт запрос |
| `data` | `null` или `[]` пока нет данных |
| `error` | `null` или текст ошибки |

Это даёт предсказуемый UI: spinner → данные / ошибка / пусто.

### Запрос с параметром

```jsx
function UserDetails({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // userId в зависимостях — при его изменении запрос перезапустится
    setLoading(true);
    setError(null);

    fetch(`https://jsonplaceholder.typicode.com/users/${userId}`)
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(setUser)
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  }, [userId]);

  // ...
}
```

### ❗ Отмена запроса при размонтировании (AbortController)

**Проблема:** пользователь быстро меняет `userId` 1 → 2 → 3. Запросы идут параллельно, и ответ на `userId=1` может прийти **последним** и перезаписать данные `userId=3`. Это называется **race condition**.

**Решение 1: AbortController**

```jsx
useEffect(() => {
  const controller = new AbortController();
  setLoading(true);

  fetch(`https://jsonplaceholder.typicode.com/users/${userId}`, {
    signal: controller.signal,
  })
    .then((res) => res.json())
    .then(setUser)
    .catch((err) => {
      // Отменённые запросы тоже «падают» — игнорируем их
      if (err.name !== 'AbortError') {
        setError(err.message);
      }
    })
    .finally(() => setLoading(false));

  // При размонтировании или смене userId — отменить запрос
  return () => controller.abort();
}, [userId]);
```

**Решение 2: флаг `ignore`**

```jsx
useEffect(() => {
  let ignore = false;

  async function load() {
    const res = await fetch(`/api/users/${userId}`);
    const data = await res.json();
    if (!ignore) setUser(data); // применяем только если эффект ещё актуален
  }

  load();

  return () => {
    ignore = true;
  };
}, [userId]);
```

Оба подхода правильные. `AbortController` лучше — реально отменяет запрос. Флаг — проще, но запрос всё равно выполняется.

---

## 5.4 🎯 Практика Части 5: Приложение с API

Сделаем **каталог пользователей** с выбором конкретного пользователя — два запроса, обработка loading/error, отмена.

### `src/api.js` — выносим запросы отдельно

```js
// Хорошая практика: вся работа с API в одном месте.
// Компоненты не должны знать про URL и структуру ответа.

const BASE = 'https://jsonplaceholder.typicode.com';

// Обёртка над fetch с проверкой res.ok и парсингом JSON.
// Экономит кучу boilerplate в каждом запросе.
export async function apiFetch(path, options = {}) {
  const res = await fetch(`${BASE}${path}`, options);
  if (!res.ok) {
    throw new Error(`HTTP ${res.status}: ${res.statusText}`);
  }
  return res.json();
}

export function getUsers(signal) {
  return apiFetch('/users', { signal });
}

export function getUserById(id, signal) {
  return apiFetch(`/users/${id}`, { signal });
}
```

### `src/App.jsx`

```jsx
import { useEffect, useState } from 'react';
import { getUsers, getUserById } from './api';

function App() {
  // ---- Список пользователей ----
  const [users, setUsers] = useState([]);
  const [usersLoading, setUsersLoading] = useState(true);
  const [usersError, setUsersError] = useState(null);

  // ---- Выбранный пользователь ----
  const [selectedId, setSelectedId] = useState(null);
  const [selectedUser, setSelectedUser] = useState(null);
  const [userLoading, setUserLoading] = useState(false);
  const [userError, setUserError] = useState(null);

  // === Эффект 1: загрузка списка пользователей ===
  useEffect(() => {
    const controller = new AbortController();

    setUsersLoading(true);
    setUsersError(null);

    getUsers(controller.signal)
      .then((data) => setUsers(data))
      .catch((err) => {
        if (err.name !== 'AbortError') setUsersError(err.message);
      })
      .finally(() => setUsersLoading(false));

    return () => controller.abort();
  }, []); // один раз при монтировании

  // === Эффект 2: загрузка деталей выбранного пользователя ===
  useEffect(() => {
    // Если ничего не выбрано — не делаем запрос
    if (selectedId === null) {
      setSelectedUser(null);
      return;
    }

    const controller = new AbortController();

    setUserLoading(true);
    setUserError(null);

    getUserById(selectedId, controller.signal)
      .then((data) => setSelectedUser(data))
      .catch((err) => {
        if (err.name !== 'AbortError') setUserError(err.message);
      })
      .finally(() => setUserLoading(false));

    return () => controller.abort();
  }, [selectedId]); // перезапуск при смене выбранного id

  // ---- Рендер списка ----
  function renderUsersList() {
    // Ранние возвраты для loading / error / empty — самый читаемый способ
    if (usersLoading) return <p>Загрузка пользователей...</p>;
    if (usersError) return <p style={{ color: 'red' }}>Ошибка: {usersError}</p>;
    if (users.length === 0) return <p>Пользователей нет</p>;

    return (
      <ul style={{ listStyle: 'none', padding: 0 }}>
        {users.map((user) => (
          <li
            key={user.id}
            onClick={() => setSelectedId(user.id)}
            style={{
              padding: '8px 12px',
              cursor: 'pointer',
              borderRadius: 6,
              marginBottom: 4,
              // Подсветка выбранного
              background: selectedId === user.id ? '#dbeafe' : 'transparent',
            }}
          >
            {user.name}
          </li>
        ))}
      </ul>
    );
  }

  // ---- Рендер деталей ----
  function renderUserDetails() {
    if (selectedId === null) {
      return <p style={{ color: '#888' }}>Выбери пользователя слева</p>;
    }
    if (userLoading) return <p>Загрузка...</p>;
    if (userError) return <p style={{ color: 'red' }}>Ошибка: {userError}</p>;
    if (!selectedUser) return null;

    const { name, email, phone, website, company, address } = selectedUser;

    return (
      <div>
        <h2>{name}</h2>
        <p><strong>Email:</strong> {email}</p>
        <p><strong>Телефон:</strong> {phone}</p>
        <p><strong>Сайт:</strong> {website}</p>
        <p><strong>Компания:</strong> {company.name}</p>
        <p><strong>Город:</strong> {address.city}</p>
      </div>
    );
  }

  return (
    <div
      style={{
        display: 'grid',
        gridTemplateColumns: '250px 1fr',
        gap: 24,
        maxWidth: 800,
        margin: '40px auto',
        padding: 20,
      }}
    >
      <aside>
        <h3>Пользователи</h3>
        {renderUsersList()}
      </aside>
      <main>
        <h3>Детали</h3>
        {renderUserDetails()}
      </main>
    </div>
  );
}

export default App;
```

**Что тренировали:**
- `useEffect` с разными зависимостями (два эффекта — `[]` и `[selectedId]`)
- Загрузка с `loading/error/data` в трёх местах
- Отмена запросов при размонтировании/смене параметра
- Ранние возвраты для состояний
- Вынос API-запросов в отдельный модуль

**✅ Чек-лист Части 5:**
- [ ] Понимаю, зачем `useEffect` и что такое «побочный эффект»
- [ ] Знаю три режима массива зависимостей (`[]`, отсутствие, `[dep]`)
- [ ] Умею писать cleanup-функцию и понимаю, когда она вызывается
- [ ] Не делаю `useEffect(async ...)` — правильно оборачиваю
- [ ] Проверяю `res.ok` после `fetch`
- [ ] Держу три состояния: loading, error, data
- [ ] Умею отменять запрос при размонтировании (AbortController)

---

# 📘 Часть 6: Формы и `useRef`

## 6.1 Два подхода к формам в React

### Контролируемые (controlled) — стандарт

Значение хранится в состоянии React.

```jsx
const [email, setEmail] = useState('');
<input value={email} onChange={(e) => setEmail(e.target.value)} />
```

**Плюсы:** всегда знаешь значение, легко валидировать/форматировать, можно сбросить программно.
**Минусы:** много state → много ререндеров (для больших форм).

### Неконтролируемые (uncontrolled)

Значение живёт **в самом DOM**, ты читаешь его через `ref`.

```jsx
const emailRef = useRef(null);
<input ref={emailRef} defaultValue="" />

// при сабмите:
const email = emailRef.current.value;
```

**Плюсы:** меньше кода для простых случаев, хорошо для файлов.
**Минусы:** сложнее валидировать «на лету», сложнее управлять.

**Правило:** используй **контролируемые** для всего, что нужно валидировать, форматировать или связывать. Используй **неконтролируемые** для файлов и очень больших форм (тогда лучше — React Hook Form).

---

## 6.2 Хук `useRef`

`useRef` возвращает **изменяемый объект** с полем `.current`.

```jsx
const myRef = useRef(initialValue);
// myRef.current — это значение
```

**Три применения:**

### 1. Ссылка на DOM-элемент

```jsx
function SearchBox() {
  const inputRef = useRef(null);

  function focus() {
    inputRef.current.focus(); // напрямую вызываем метод DOM-элемента
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={focus}>Сфокусировать</button>
    </>
  );
}
```

### 2. Хранение «теневого» значения между рендерами

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const renders = useRef(0); // не вызывает ререндер при изменении

  renders.current += 1; // мутируем спокойно

  return <p>Значение: {count}, рендеров: {renders.current}</p>;
}
```

**Ключевое отличие от `useState`:**
- `useState` → изменение вызывает **ререндер**
- `useRef` → изменение **не вызывает ререндер**

### 3. Предыдущее значение

```jsx
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value; // обновляем ПОСЛЕ рендера
  }, [value]);
  return ref.current; // возвращаем значение из ПРОШЛОГО рендера
}
```

---

## 6.3 Формы с несколькими полями

### Паттерн «объект в state»

```jsx
function RegistrationForm() {
  const [form, setForm] = useState({
    name: '',
    email: '',
    password: '',
    age: '',
    agree: false,
  });

  // Один обработчик для ВСЕХ полей
  function handleChange(e) {
    const { name, value, type, checked } = e.target;

    setForm((prev) => ({
      ...prev,
      [name]: type === 'checkbox' ? checked : value,
    }));
  }

  return (
    <form>
      <input name="name" value={form.name} onChange={handleChange} />
      <input name="email" value={form.email} onChange={handleChange} />
      <input name="password" type="password" value={form.password} onChange={handleChange} />
      <input name="age" type="number" value={form.age} onChange={handleChange} />
      <input
        name="agree"
        type="checkbox"
        checked={form.agree}
        onChange={handleChange}
      />
    </form>
  );
}
```

**Ключевой приём:** атрибут `name` в инпуте должен совпадать с ключом в state. Тогда `[name]: value` через **computed property** обновляет нужное поле.

---

## 6.4 Валидация

### Минимальный паттерн: ошибки в state

```jsx
function Form() {
  const [email, setEmail] = useState('');
  const [error, setError] = useState('');

  function handleSubmit(e) {
    e.preventDefault();

    // Валидация
    if (!email.includes('@')) {
      setError('Некорректный email');
      return;
    }

    setError('');
    // отправка
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
      {error && <p style={{ color: 'red' }}>{error}</p>}
      <button>Отправить</button>
    </form>
  );
}
```

### Валидация на лету vs при сабмите

**Вариант 1: при сабмите** — ошибки появляются только после нажатия «Отправить».

**Вариант 2: на лету (при вводе)** — ошибка исчезает, как только пользователь исправил. Плохо — пользователь видит «Некорректный email» уже после первой буквы.

**Вариант 3 (лучший): «touched» — ошибки только для полей, которые пользователь уже посещал.**

```jsx
const [touched, setTouched] = useState({});

function handleBlur(e) {
  setTouched((prev) => ({ ...prev, [e.target.name]: true }));
}

// ошибка показывается только если поле "тронуто"
{touched.email && errors.email && <p style={{ color: 'red' }}>{errors.email}</p>}
```

### «Ошибки — это вычисляемое»

Не храни `errors` в state — вычисляй из значений. Иначе нужно синхронизировать два state.

```jsx
// ✅ Производные ошибки — вычисляются при каждом рендере
const errors = {
  name: form.name.length < 2 ? 'Слишком короткое имя' : '',
  email: !form.email.includes('@') ? 'Некорректный email' : '',
  password: form.password.length < 6 ? 'Минимум 6 символов' : '',
};

const isValid = Object.values(errors).every((e) => !e);
```

---

## 6.5 🎯 Практика Части 6: Форма регистрации

Сделаем полноценную форму с валидацией, «touched», показом пароля, автофокусом и сбросом.

### Финальная структура

```
src/
├── components/
│   └── RegistrationForm.jsx
├── utils/
│   └── validation.js
└── App.jsx
```

### `src/utils/validation.js` — правила валидации

```js
// Выносим валидацию отдельно: чистая функция, легко тестировать.

export function validateForm({ name, email, password, confirmPassword, age, agree }) {
  const errors = {};

  // Имя: 2+ символа, только буквы и пробелы
  if (!name.trim()) {
    errors.name = 'Введите имя';
  } else if (name.trim().length < 2) {
    errors.name = 'Минимум 2 символа';
  } else if (!/^[a-zA-Zа-яА-ЯёЁ\s]+$/.test(name)) {
    errors.name = 'Только буквы и пробелы';
  }

  // Email: простой regex
  if (!email.trim()) {
    errors.email = 'Введите email';
  } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    errors.email = 'Некорректный email';
  }

  // Пароль: 6+ символов, хотя бы одна цифра и одна буква
  if (!password) {
    errors.password = 'Введите пароль';
  } else if (password.length < 6) {
    errors.password = 'Минимум 6 символов';
  } else if (!/\d/.test(password) || !/[a-zA-Z]/.test(password)) {
    errors.password = 'Должен содержать буквы и цифры';
  }

  // Подтверждение пароля
  if (confirmPassword !== password) {
    errors.confirmPassword = 'Пароли не совпадают';
  }

  // Возраст: 18–120
  if (!age) {
    errors.age = 'Введите возраст';
  } else if (Number(age) < 18) {
    errors.age = 'Только с 18 лет';
  } else if (Number(age) > 120) {
    errors.age = 'Слишком большой возраст';
  }

  // Согласие
  if (!agree) {
    errors.agree = 'Нужно согласие';
  }

  return errors;
}
```

### `src/components/RegistrationForm.jsx`

```jsx
import { useRef, useState } from 'react';
import { validateForm } from '../utils/validation';

function RegistrationForm({ onSuccess }) {
  // ---- Одно состояние на всю форму ----
  const [form, setForm] = useState({
    name: '',
    email: '',
    password: '',
    confirmPassword: '',
    age: '',
    agree: false,
  });

  // ---- Какие поля пользователь уже "трогал" ----
  // Ошибки показываем только для тронутых полей — это лучший UX.
  const [touched, setTouched] = useState({});

  // ---- Показ пароля ----
  const [showPassword, setShowPassword] = useState(false);

  // ---- Ref на первое поле для автофокуса ----
  const nameRef = useRef(null);

  // ---- Вычисляем ошибки на каждом рендере ----
  // НЕ храним ошибки в state — они производные от form
  const errors = validateForm(form);
  const isValid = Object.keys(errors).length === 0;

  // ---- Обработчики ----

  function handleChange(e) {
    const { name, value, type, checked } = e.target;
    setForm((prev) => ({
      ...prev,
      [name]: type === 'checkbox' ? checked : value,
    }));
  }

  function handleBlur(e) {
    // Помечаем поле как "тронутое" — теперь его ошибки можно показывать
    setTouched((prev) => ({ ...prev, [e.target.name]: true }));
  }

  function handleSubmit(e) {
    e.preventDefault();

    // Помечаем ВСЕ поля тронутыми — чтобы показать все ошибки
    const allTouched = Object.keys(form).reduce(
      (acc, key) => ({ ...acc, [key]: true }),
      {}
    );
    setTouched(allTouched);

    if (!isValid) {
      // Фокусируем первое невалидное поле
      const firstErrorField = Object.keys(errors)[0];
      document.querySelector(`[name="${firstErrorField}"]`)?.focus();
      return;
    }

    // Всё валидно — отправляем
    onSuccess?.(form);

    // Сброс формы
    setForm({
      name: '',
      email: '',
      password: '',
      confirmPassword: '',
      age: '',
      agree: false,
    });
    setTouched({});
    setShowPassword(false);

    // Возвращаем фокус на первое поле
    nameRef.current?.focus();
  }

  // ---- Хелпер для рендера поля ошибки ----
  function renderError(field) {
    if (!touched[field] || !errors[field]) return null;
    return (
      <p style={{ color: '#ef4444', fontSize: 13, margin: '4px 0 0' }}>
        {errors[field]}
      </p>
    );
  }

  // ---- Стили полей ----
  const fieldStyle = (field) => ({
    width: '100%',
    padding: 10,
    borderRadius: 8,
    border: `1px solid ${
      touched[field] && errors[field] ? '#ef4444' : '#ddd'
    }`,
    fontSize: 14,
  });

  return (
    <form
      onSubmit={handleSubmit}
      style={{ display: 'flex', flexDirection: 'column', gap: 14 }}
      noValidate // отключаем браузерную валидацию — у нас своя
    >
      <div>
        <label htmlFor="name">Имя</label>
        <input
          id="name"
          name="name"
          ref={nameRef}
          autoFocus
          value={form.name}
          onChange={handleChange}
          onBlur={handleBlur}
          style={fieldStyle('name')}
        />
        {renderError('name')}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          name="email"
          type="email"
          value={form.email}
          onChange={handleChange}
          onBlur={handleBlur}
          style={fieldStyle('email')}
        />
        {renderError('email')}
      </div>

      <div>
        <label htmlFor="password">Пароль</label>
        <div style={{ display: 'flex', gap: 6 }}>
          <input
            id="password"
            name="password"
            type={showPassword ? 'text' : 'password'}
            value={form.password}
            onChange={handleChange}
            onBlur={handleBlur}
            style={fieldStyle('password')}
          />
          <button
            type="button"
            onClick={() => setShowPassword((v) => !v)}
            style={{ padding: '0 12px', cursor: 'pointer' }}
          >
            {showPassword ? '🙈' : '👁️'}
          </button>
        </div>
        {renderError('password')}
      </div>

      <div>
        <label htmlFor="confirmPassword">Повтор пароля</label>
        <input
          id="confirmPassword"
          name="confirmPassword"
          type="password"
          value={form.confirmPassword}
          onChange={handleChange}
          onBlur={handleBlur}
          style={fieldStyle('confirmPassword')}
        />
        {renderError('confirmPassword')}
      </div>

      <div>
        <label htmlFor="age">Возраст</label>
        <input
          id="age"
          name="age"
          type="number"
          value={form.age}
          onChange={handleChange}
          onBlur={handleBlur}
          style={fieldStyle('age')}
        />
        {renderError('age')}
      </div>

      <div>
        <label style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
          <input
            name="agree"
            type="checkbox"
            checked={form.agree}
            onChange={handleChange}
            onBlur={handleBlur}
          />
          Согласен с условиями
        </label>
        {renderError('agree')}
      </div>

      <button
        type="submit"
        disabled={!isValid}
        style={{
          padding: 12,
          background: isValid ? '#3b82f6' : '#9ca3af',
          color: 'white',
          border: 'none',
          borderRadius: 8,
          fontSize: 15,
          fontWeight: 600,
          cursor: isValid ? 'pointer' : 'not-allowed',
        }}
      >
        Зарегистрироваться
      </button>
    </form>
  );
}

export default RegistrationForm;
```

### `src/App.jsx`

```jsx
import { useState } from 'react';
import RegistrationForm from './components/RegistrationForm';

function App() {
  const [registered, setRegistered] = useState(null);

  function handleSuccess(data) {
    setRegistered(data);
  }

  return (
    <div style={{ maxWidth: 450, margin: '40px auto', padding: 20 }}>
      <h1>Регистрация</h1>

      {registered ? (
        <div style={{ padding: 16, background: '#dcfce7', borderRadius: 8 }}>
          <p>✅ Спасибо, {registered.name}!</p>
          <button onClick={() => setRegistered(null)}>
            Зарегистрировать ещё
          </button>
        </div>
      ) : (
        <RegistrationForm onSuccess={handleSuccess} />
      )}
    </div>
  );
}

export default App;
```

---

## 🔍 Разбор ключевых решений

### 1. Ошибки **не** в state

```jsx
const errors = validateForm(form);  // вычисляется на каждом рендере
```

Если бы держали `errors` в `useState`, пришлось бы вызывать `setErrors` при каждом изменении формы. Легко забыть → баг. Вычисление — всегда актуально.

### 2. «Touched» — правильный UX

- Пользователь вводит первую букву email → **не показываем** «некорректный email»
- Пользователь уходит с поля (`onBlur`) → **теперь показываем** ошибку
- При сабмите → **показываем все** ошибки

Без «touched» пользователь видит ошибки сразу при загрузке формы — это раздражает.

### 3. `useRef` для автофокуса

```jsx
const nameRef = useRef(null);
// ...
<input ref={nameRef} autoFocus />
// после сабмита:
nameRef.current?.focus();
```

`useRef` даёт прямой доступ к DOM-элементу. Без ref фокус пришлось бы делать через `document.querySelector` — некрасиво.

### 4. `showPassword` — отдельное состояние

`type` инпута управляется состоянием → React контролирует его. Переключение `password` ↔ `text` — самый частый кейс `useState` для UI-флагов.

---

## 🏋️ Упражнения для закрепления

### Часть 5
1. **Приложение «Факты о кошках»**: загрузи случайный факт с `https://catfact.ninja/fact`, кнопка «Ещё» загружает новый. Обработай loading/error.
2. **Погода**: возьми `https://open-meteo.com/en/docs` (без ключа). Инпут для города → координаты → прогноз. Покажи текущую температуру.
3. **Пагинация**: список постов с `https://jsonplaceholder.typicode.com/posts?_page=1&_limit=10`. Кнопки «Назад/Вперёд» меняют страницу.
4. **Поиск с debounce**: инпут поиска, запрос отправляется через 500 мс после последнего ввода (через `setTimeout` в `useEffect` с cleanup).

### День 6
5. **Форма отзыва**: имя, оценка (1–5 звёзд), текст (min 10 символов). Валидация, disabled кнопки, сброс.
6. **Форма с зависимостями**: поля «страна» и «город» — город валидируется по-разному в зависимости от страны.
7. **Форма с фокусом**: после неудачного сабмита фокусируется первое невалидное поле (уже есть — сделай сам в упражнении).
8. **Файловый инпут**: `<input type="file" ref={...} />`, показывай имя выбранного файла (это кейс, где `useRef` — стандарт).
9. **Форма с `useRef` для таймера**: если пользователь не активен в форме 30 сек — покажи подсказку «Всё ещё с нами?». Используй `useEffect` + `setTimeout` + cleanup + `useRef` для хранения таймера.
10. **Многошаговая форма**: шаг 1 (личное), шаг 2 (адрес), шаг 3 (подтверждение). Валидация на каждом шаге, кнопки «Назад/Далее». Состояние всей формы — на верхнем уровне.

---

## 📋 Чек-лист самопроверки

**Часть 5:**
- [ ] Понимаю, почему `useEffect(async ...)` — ошибка и как это обойти
- [ ] Знаю, когда эффект срабатывает при `[]`, без массива и `[dep]`
- [ ] Пишу cleanup-функции и понимаю, зачем они
- [ ] Проверяю `res.ok` после `fetch`
- [ ] Всегда держу три состояния: loading, error, data
- [ ] Отменяю запросы через `AbortController` при смене параметра / размонтировании
- [ ] Выношу работу с API в отдельный модуль

**Часть 6:**
- [ ] Понимаю разницу controlled vs uncontrolled
- [ ] Умею использовать `useRef` для фокуса
- [ ] Знаю, что `useRef.current` не вызывает ререндер
- [ ] Один обработчик `handleChange` с `name`-атрибутом для всех полей
- [ ] Вычисляю ошибки, а не храню их в state
- [ ] Использую «touched» для правильного UX с валидацией
- [ ] Умею показывать/скрывать пароль через `type`
- [ ] Делаю автофокус и фокусировку на ошибку через ref

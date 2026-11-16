
## 📝 Домашнее задание: «Мини-заметки»

**Цель:** Собрать одно работающее приложение с навигацией, загрузкой данных с локального API, формой создания и удаления, а также продакшен-сборкой.

### 📌 Что нужно сделать

**1. Подготовьте `json-server`**
Создайте в корне проекта файл `db.json`:
```json
{
  "notes": [
    { "id": 1, "title": "Первая заметка", "text": "Купить молоко" },
    { "id": 2, "title": "Вторая заметка", "text": "Позвонить маме" }
  ]
}
```
Запустите сервер:
```bash
npx json-server --watch db.json --port 3000
```
Убедитесь, что `http://localhost:3000/notes` открывается в браузере и показывает JSON-массив.

---

**2. Роутинг**

Создайте три Standalone-компонента:
- `home` — главная страница с текстом «Добро пожаловать в Мини-заметки».
- `notes-list` — список заметок.
- `note-form` — форма создания заметки.

В `app.routes.ts` настройте:
```typescript
export const routes: Routes = [
  { path: '', component: HomeComponent, title: 'Главная' },
  { path: 'notes', component: NotesListComponent, title: 'Заметки' },
  { path: 'create', component: NoteFormComponent, title: 'Создать заметку' },
  { path: '**', redirectTo: '' }
];
```
Подключите `provideRouter(routes)` в `main.ts`. В `app.component.html` сделайте навигацию:
```html
<nav>
  <a routerLink="/">Главная</a>
  <a routerLink="/notes">Заметки</a>
  <a routerLink="/create">Создать</a>
</nav>
<router-outlet></router-outlet>
```
Не забудьте импортировать `RouterOutlet` и `RouterLink` в `imports` корневого компонента.

---

**3. Сервис для работы с API**

Создайте сервис `note` (`ng generate service note`). В нём:
```typescript
export interface Note {
  id: number;
  title: string;
  text: string;
}

@Injectable({ providedIn: 'root' })
export class NoteService {
  private http = inject(HttpClient);
  private apiUrl = 'http://localhost:3000/notes';

  getNotes(): Observable<Note[]> {
    return this.http.get<Note[]>(this.apiUrl);
  }

  addNote(note: { title: string; text: string }): Observable<Note> {
    return this.http.post<Note>(this.apiUrl, note);
  }

  deleteNote(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```
Убедитесь, что в `main.ts` добавлен `provideHttpClient()`.

---

**4. Список заметок с загрузкой и удалением**

В `notes-list.component.ts`:
- Внедрите `NoteService`.
- Заведите `notes: Note[] = []`, `isLoading = true`, `errorMessage = ''`.
- В `ngOnInit` подпишитесь на `getNotes()`:
  - `next` → присвойте данные, `isLoading = false`.
  - `error` → покажите сообщение, `isLoading = false`.
- Добавьте метод `remove(id: number)`, который вызывает `deleteNote(id).subscribe(() => ...)` и удаляет заметку из локального массива (или перезагружает список).

В шаблоне:
- `@if (isLoading)` → «Загрузка...»
- `@if (errorMessage)` → текст ошибки
- `@for (note of notes; track note.id)` → карточка с `title`, `text` и кнопкой «Удалить» `(click)="remove(note.id)"`
- `@empty` → «Заметок нет»

---

**5. Форма создания заметки**

В `note-form.component.ts`:
- Через `FormBuilder` создайте `FormGroup`:
  - `title`: `['', [Validators.required, Validators.minLength(3)]]`
  - `text`: `['', [Validators.required, Validators.minLength(5)]]`
- Геттеры `get title()` и `get text()` для доступа к контролам.
- В `onSubmit()`:
  - Если форма валидна → вызовите `addNote(this.form.value).subscribe(...)`.
    - В `next` → `this.form.reset()`, `this.successMessage = 'Заметка создана'`.
    - В `error` → `this.errorMessage = 'Не удалось создать заметку'`.
  - Если невалидна → `this.form.markAllAsTouched()`.

В шаблоне:
- `[formGroup]="noteForm"` и `(ngSubmit)="onSubmit()"`.
- Поля с `formControlName="title"` и `formControlName="text"`.
- Под каждым полем — ошибки через `@if (title?.invalid && title?.touched)` с сообщениями для `required` и `minlength`.
- Кнопка `[disabled]="noteForm.invalid"`.
- `@if (successMessage)` → зелёное сообщение об успехе.

---

**6. Сборка**

Когда всё работает в режиме разработки, выполните:
```bash
ng build --configuration production
```
Убедитесь, что сборка проходит без ошибок и в папке `dist/` появились файлы. Запустите собранную версию локально:
```bash
npx http-server dist/my-app/browser
```
(путь может отличаться — проверьте содержимое `dist/`).

---

###  Критерий готовности (проверьте себя)

| Что проверить | Ожидаемый результат |
|---|---|
| Навигация | Три ссылки переключают страницы без перезагрузки, `**` редиректит на главную |
| Список заметок | При переходе на `/notes` виден индикатор загрузки, затем список из `db.json` |
| Ошибка сети | Остановите `json-server` и обновите `/notes` — показывается сообщение об ошибке, а не бесконечная загрузка |
| Форма | Кнопка неактивна, пока поля не заполнены корректно; при касании пустого поля появляется ошибка |
| Создание | После отправки формы заметка появляется в `db.json` и в списке на `/notes` |
| Удаление | Кнопка «Удалить» убирает заметку из списка и из `db.json` |
| Сборка | `ng build --configuration production` проходит без ошибок, `dist/` запускается локально |

# Подгрузку сервера через node.js

``` bash
npm install -g json-server
```

# В корневом файле проекта создание файла `db.json`

```json
{
  "posts": [
    { "id": 1, "title": "Первый пост", "body": "Это тестовое тело поста" },
    { "id": 2, "title": "Второй пост", "body": "Ещё одно тело для примера" }
  ]
}

```


# Запуск сервера

```bash
json-server --watch db.json --port 3000
```


# Ссылка к постам

```
http://localhost:3000/posts
```

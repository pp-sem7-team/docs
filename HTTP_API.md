# HTTP API Contract

## Общая информация

### Архитектура и Base URL

API предназначен для использования веб-клиентом.

Base URL задаётся конфигурацией окружения. Все эндпоинты указываются относительно Base URL.

### Формат данных

Основной формат передачи данных - JSON.

Content-Type запросов и ответов: `application/json`

Именование полей - `snake_case`. Идентификаторы - UUID. Даты и время - строки в формате ISO 8601 (UTC), например `2026-01-01T12:00:00Z`.

### Авторизация

JWT: пара `access_token` и `refresh_token`. Токен передаётся в заголовке `Authorization: Bearer access_token`.

Без токена доступны только эндпоинты `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout` и `/health*`.

### Асинхронная генерация

Анализ, коммит и описание PR генерируются асинхронно. Такие ресурсы имеют поле `status`: `pending` (выполняется), `done` (готово), `failed` (ошибка). Запрос на генерацию сразу возвращает `202` со статусом `pending`, результат клиент получает опросом соответствующего `GET` эндпоинта.

### Формат ошибок

```json
{
    "error": {
        "code": "string",
        "message": "string"
    }
}
```

### Коды ответа

API использует стандартные HTTP-коды ответа:

- 200 OK - запрос выполнен успешно
- 201 Created - ресурс создан
- 202 Accepted - запрос принят, обработка выполняется асинхронно
- 204 No Content - успешное выполнение без тела ответа
- 400 Bad Request - некорректный запрос: невалидное тело или параметры, пустые или недопустимые значения полей, некорректный diff
- 401 Unauthorized - токен отсутствует, недействителен или истёк; неверные email или пароль; недействительный или отозванный `refresh_token`
- 404 Not Found - ресурс не найден или принадлежит другому пользователю
- 409 Conflict - конфликт состояния ресурса:
  - ресурс с таким email или именем уже существует (пользователь, рабочая область, ветка)
  - ветка закрыта (`closed`), добавить в неё изменение нельзя
  - ресурс не в подходящем статусе генерации: редактирование или повторная генерация при `pending`, редактирование при `failed`, генерация коммита или PR, пока анализ не в статусе `done`
  - генерация PR для ветки без изменений
- 413 Payload Too Large - превышен максимальный размер diff
- 500 Internal Server Error - внутренняя ошибка сервера
- 503 Service Unavailable - недоступны инфраструктурные зависимости

Максимальный размер diff задаётся конфигурацией.

## Структуры данных

### User
```json
{
    "id": "UUID",
    "email": "user@example.com",
    "created_at": "2026-01-01T12:00:00Z",
    "updated_at": "2026-01-01T12:00:00Z"
}
```

### Workspace
```json
{
    "id": "UUID",
    "name": "string",
    "created_at": "2026-01-01T12:00:00Z",
    "updated_at": "2026-01-01T12:00:00Z"
}
```

### Branch
```json
{
    "id": "UUID",
    "workspace_id": "UUID",
    "name": "string",
    "status": "active" | "closed",
    "created_at": "2026-01-01T12:00:00Z",
    "updated_at": "2026-01-01T12:00:00Z"
}
```

### Change
```json
{
    "id": "UUID",
    "branch_id": "UUID",
    "sequence_number": 1,
    "analysis": AnalysisResult,
    "commit": Commit,
    "created_at": "2026-01-01T12:00:00Z"
}
```

### AnalysisResult
`structured_changes` - JSON с результатом анализа. Равно `null`, пока `status` не станет `done`.

```json
{
    "id": "UUID",
    "structured_changes": json | null,
    "status": "pending" | "done" | "failed",
    "created_at": "2026-01-01T12:00:00Z",
    "updated_at": "2026-01-01T12:00:00Z"
}
```

### Commit
Поля `type`, `scope` и `message` равны `null`, пока `status` не станет `done`. `scope` может остаться `null` и после генерации.

```json
{
    "id": "UUID",
    "type": "feat" | "fix" | "refactor" | "perf" | "docs" | "test" | "style" | "chore" | null,
    "scope": "string" | null,
    "message": "string" | null,
    "is_edited_by_user": false,
    "status": "pending" | "done" | "failed",
    "created_at": "2026-01-01T12:00:00Z",
    "updated_at": "2026-01-01T12:00:00Z"
}
```

### PullRequest
Поля `title` и `description` равны `null`, пока `status` не станет `done`.

```json
{
    "id": "UUID",
    "branch_id": "UUID",
    "title": "string" | null,
    "description": "string" | null,
    "is_edited_by_user": false,
    "status": "pending" | "done" | "failed",
    "created_at": "2026-01-01T12:00:00Z",
    "updated_at": "2026-01-01T12:00:00Z"
}
```

## Эндпоинты

### Список эндпоинтов
[`POST   /auth/register`](#post-authregister)\
[`POST   /auth/login`](#post-authlogin)\
[`POST   /auth/refresh`](#post-authrefresh)\
[`POST   /auth/logout`](#post-authlogout)

[`GET    /users/me`](#get-usersme)\
[`PATCH  /users/me/password`](#patch-usersmepassword)

[`POST   /workspaces`](#post-workspaces)\
[`GET    /workspaces`](#get-workspaces)\
[`GET    /workspaces/{workspace_id}`](#get-workspacesworkspace_id)\
[`PATCH  /workspaces/{workspace_id}`](#patch-workspacesworkspace_id)\
[`DELETE /workspaces/{workspace_id}`](#delete-workspacesworkspace_id)

[`POST   /workspaces/{workspace_id}/branches`](#post-workspacesworkspace_idbranches)\
[`GET    /workspaces/{workspace_id}/branches`](#get-workspacesworkspace_idbranches)\
[`GET    /branches/{branch_id}`](#get-branchesbranch_id)\
[`PATCH  /branches/{branch_id}`](#patch-branchesbranch_id)\
[`DELETE /branches/{branch_id}`](#delete-branchesbranch_id)

[`POST   /branches/{branch_id}/changes`](#post-branchesbranch_idchanges)\
[`GET    /branches/{branch_id}/changes`](#get-branchesbranch_idchanges)\
[`GET    /changes/{change_id}`](#get-changeschange_id)

[`POST   /changes/{change_id}/commit`](#post-changeschange_idcommit)\
[`PATCH  /changes/{change_id}/commit`](#patch-changeschange_idcommit)

[`POST   /branches/{branch_id}/pull-request`](#post-branchesbranch_idpull-request)\
[`GET    /branches/{branch_id}/pull-request`](#get-branchesbranch_idpull-request)\
[`PATCH  /branches/{branch_id}/pull-request`](#patch-branchesbranch_idpull-request)

[`GET    /health`](#get-health)\
[`GET    /health/db`](#get-healthdb)

---

## Auth

### [POST /auth/register](#список-эндпоинтов)
Регистрация нового пользователя.

#### Request
```json
{
    "email": "user@example.com",
    "password": "string"
}
```

#### Response
```json
201 Created
{
    "access_token": "string",
    "refresh_token": "string"
}
```

### [POST /auth/login](#список-эндпоинтов)
Аутентификация пользователя.

#### Request
```json
{
    "email": "user@example.com",
    "password": "string"
}
```

#### Response
```json
200 OK
{
    "access_token": "string",
    "refresh_token": "string"
}
```

### [POST /auth/refresh](#список-эндпоинтов)
Обновление пары токенов. Использованный `refresh_token` отзывается.

#### Request
```json
{
    "refresh_token": "string"
}
```

#### Response
```json
200 OK
{
    "access_token": "string",
    "refresh_token": "string"
}
```

### [POST /auth/logout](#список-эндпоинтов)
Выход из системы. Переданный `refresh_token` отзывается.

#### Request
```json
{
    "refresh_token": "string"
}
```

#### Response
```json
204 No Content
```

---

## Users

### [GET /users/me](#список-эндпоинтов)
Получение профиля текущего пользователя.

#### Request
```json
Authorization: Bearer access_token
```

#### Response
```json
200 OK
User
```

### [PATCH /users/me/password](#список-эндпоинтов)
Смена пароля текущего пользователя.

#### Request
```json
Authorization: Bearer access_token
{
    "current_password": "string",
    "new_password": "string"
}
```

#### Response
```json
204 No Content
```

---

## Workspaces

### [POST /workspaces](#список-эндпоинтов)
Создание рабочей области.

#### Request
```json
Authorization: Bearer access_token
{
    "name": "string"
}
```

#### Response
```json
201 Created
Workspace
```

### [GET /workspaces](#список-эндпоинтов)
Список рабочих областей текущего пользователя.

#### Request
```json
Authorization: Bearer access_token
Query:
    offset (int, default: 0)
    limit  (int, default: 10, max: 100)
```

#### Response
```json
200 OK
{
    "items": [Workspace],
    "limit": 0,
    "offset": 0,
    "total": 0
}
```

### [GET /workspaces/{workspace_id}](#список-эндпоинтов)
Получение рабочей области.

#### Request
```json
Authorization: Bearer access_token
```

#### Response
```json
200 OK
Workspace
```

### [PATCH /workspaces/{workspace_id}](#список-эндпоинтов)
Переименование рабочей области.

#### Request
```json
Authorization: Bearer access_token
{
    "name": "string"
}
```

#### Response
```json
200 OK
Workspace
```

### [DELETE /workspaces/{workspace_id}](#список-эндпоинтов)
Удаление рабочей области вместе со всеми ветками, изменениями, коммитами и PR.

#### Request
```json
Authorization: Bearer access_token
```

#### Response
```json
204 No Content
```

---

## Branches

### [POST /workspaces/{workspace_id}/branches](#список-эндпоинтов)
Создание ветки в рабочей области.

#### Request
```json
Authorization: Bearer access_token
{
    "name": "feature/user-search"
}
```

#### Response
```json
201 Created
Branch
```

### [GET /workspaces/{workspace_id}/branches](#список-эндпоинтов)
Список веток рабочей области.

#### Request
```json
Authorization: Bearer access_token
Query:
    offset (int, default: 0)
    limit  (int, default: 10, max: 100)
    status (string, active | closed, optional)
```

#### Response
```json
200 OK
{
    "items": [Branch],
    "limit": 0,
    "offset": 0,
    "total": 0
}
```

### [GET /branches/{branch_id}](#список-эндпоинтов)
Получение ветки.

#### Request
```json
Authorization: Bearer access_token
```

#### Response
```json
200 OK
Branch
```

### [PATCH /branches/{branch_id}](#список-эндпоинтов)
Изменение имени и/или статуса ветки.

#### Request
```json
Authorization: Bearer access_token
{
    "name": "string",
    "status": "active" | "closed"
}
```

#### Response
```json
200 OK
Branch
```

### [DELETE /branches/{branch_id}](#список-эндпоинтов)
Удаление ветки вместе со всеми изменениями, коммитами и PR.

#### Request
```json
Authorization: Bearer access_token
```

#### Response
```json
204 No Content
```

---

## Changes

### [POST /branches/{branch_id}/changes](#список-эндпоинтов)
Добавление изменения в ветку. Запускает анализ и генерацию сообщения коммита.

#### Request
```json
Authorization: Bearer access_token
{
    "raw_diff": "string"
}
```

#### Response
```json
202 Accepted
Change
```

### [GET /branches/{branch_id}/changes](#список-эндпоинтов)
История изменений ветки, отсортированная по `sequence_number`.

#### Request
```json
Authorization: Bearer access_token
Query:
    offset (int, default: 0)
    limit  (int, default: 10, max: 100)
```

#### Response
```json
200 OK
{
    "items": [Change],
    "limit": 0,
    "offset": 0,
    "total": 0
}
```

### [GET /changes/{change_id}](#список-эндпоинтов)
Получение изменения.

#### Request
```json
Authorization: Bearer access_token
```

#### Response
```json
200 OK
Change
```

---

## Commits

### [POST /changes/{change_id}/commit](#список-эндпоинтов)
Генерация сообщения коммита. Повторный вызов перезаписывает результат и сбрасывает `is_edited_by_user`. Доступно, только если анализ в статусе `done`.

#### Request
```json
Authorization: Bearer access_token
```

#### Response
```json
202 Accepted
Commit
```

### [PATCH /changes/{change_id}/commit](#список-эндпоинтов)
Ручное редактирование коммита. `is_edited_by_user` становится `true`. Доступно только при `status: "done"`.

#### Request
```json
Authorization: Bearer access_token
{
    "type": "feat" | "fix" | "refactor" | "perf" | "docs" | "test" | "style" | "chore",
    "scope": "string" | null,
    "message": "string"
}
```

#### Response
```json
200 OK
Commit
```

---

## Pull Requests

### [POST /branches/{branch_id}/pull-request](#список-эндпоинтов)
Генерация описания PR по всем изменениям ветки. Если PR уже есть, перезаписывает его и сбрасывает `is_edited_by_user`. Доступно, только если в ветке есть изменения и анализ всех изменений в статусе `done`.

#### Request
```json
Authorization: Bearer access_token
```

#### Response
```json
202 Accepted
PullRequest
```

### [GET /branches/{branch_id}/pull-request](#список-эндпоинтов)
Получение PR ветки.

#### Request
```json
Authorization: Bearer access_token
```

#### Response
```json
200 OK
PullRequest
```

### [PATCH /branches/{branch_id}/pull-request](#список-эндпоинтов)
Ручное редактирование PR. `is_edited_by_user` становится `true`. Доступно только при `status: "done"`.

#### Request
```json
Authorization: Bearer access_token
{
    "title": "string",
    "description": "string"
}
```

#### Response
```json
200 OK
PullRequest
```

---

## Health

### [GET /health](#список-эндпоинтов)
Проверка работоспособности сервиса.

#### Response
```json
200 OK
```

### [GET /health/db](#список-эндпоинтов)
Проверка доступности базы данных.

#### Response
```json
200 OK
```

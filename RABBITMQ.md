# RabbitMQ Message Contract

## Общая информация

### Участники

- `backend` - продюсер команд, консьюмер результатов.
- `ai-service` - консьюмер команд, продюсер результатов.

### Поток сообщений

Команды попадают в брокер через `outbox`: запись в `outbox` и изменение данных делаются в одной транзакции БД.

#### POST /branches/{branch_id}/changes

1. Создаются Change, AnalysisResult и Commit (у AnalysisResult и Commit статус `pending`), в `outbox` добавляется команда `analysis.request`.
2. Приходит `analysis.result`:
   - `status: done` - AnalysisResult получает `done` и `structured_changes`, в `outbox` добавляется команда `commit.request`. Commit остаётся `pending`.
   - `status: failed` - AnalysisResult и Commit получают `failed`, команда `commit.request` не создаётся.
3. Приходит `commit.result` - обновляется Commit.

#### POST /changes/{change_id}/commit

1. Commit получает `pending`, в `outbox` добавляется команда `commit.request` (повторная генерация).
2. Приходит `commit.result` - обновляется Commit.

#### POST /branches/{branch_id}/pull-request

1. PullRequest получает `pending`, в `outbox` добавляется команда `pull_request.request`.
2. Приходит `pull_request.result` - обновляется PullRequest.

### AMQP-свойства

| Свойство | Значение |
|---|---|
| `content_type` | `application/json` |
| `content_encoding` | `utf-8` |
| `delivery_mode` | `2` (persistent) |
| `message_id` | UUID, уникален для каждого сообщения |
| `correlation_id` | у команды равен её `message_id`; у результата равен `message_id` команды, на которую он отвечает |
| `type` | имя сообщения, совпадает с routing key (например, `analysis.request`) |
| `timestamp` | время публикации |
| `app_id` | `backend` или `ai-service` |

### Гарантии доставки

- At-least-once. Дубликаты возможны, консьюмеры обязаны быть идемпотентными.
- Продюсеры используют publisher confirms и флаг `mandatory`. Возврат сообщения брокером (`basic.return`) считается ошибкой публикации.
- Консьюмеры работают с manual ack, `prefetch` задаётся конфигурацией. Ack отправляется только после полной обработки, для `ai-service` - после публикации результата, подтверждённой брокером.

### Идемпотентность и устаревшие результаты

`backend` хранит `message_id` последней отправленной команды для каждого ресурса (AnalysisResult, Commit, PullRequest). Результат применяется, только если одновременно:

1. ресурс существует (не удалён вместе с веткой или рабочей областью);
2. ресурс в статусе `pending`;
3. `correlation_id` результата равен сохранённому `message_id` команды.

Иначе результат подтверждается (ack) и отбрасывается.

### Ошибки и повторы

| Ситуация | Поведение |
|---|---|
| Ожидаемая ошибка генерации (недоступен провайдер, невалидный ответ модели) | `ai-service` делает ограниченное число внутренних повторов, затем публикует результат со `status: failed` и делает ack |
| Неожиданный сбой `ai-service` при обработке | `nack` с `requeue=true`. После `x-delivery-limit` попыток сообщение уходит в DLQ |
| `backend` не смог обработать результат (например, БД недоступна) | `nack` с `requeue=true`. После `x-delivery-limit` попыток сообщение уходит в DLQ |
| Сообщение в DLQ | Обрабатывается вручную (алертинг), консьюмеров у DLQ нет |

Чтобы ресурс не завис в `pending`, если сообщение потерялось или попало в DLQ, `backend` переводит ресурсы в `failed` по таймауту генерации. Значение таймаута задаётся конфигурацией. Для Commit таймаут отсчитывается от отправки `commit.request`. Если по таймауту анализ получил `failed`, Commit тоже получает `failed`.

## Топология

Топологию объявляют оба сервиса при старте.

Все exchanges: тип `direct`, `durable`.

| Exchange | Назначение | Продюсер | Консьюмер |
|---|---|---|---|
| `generation.commands` | Команды на генерацию | `backend` | `ai-service` |
| `generation.results` | Результаты генерации | `ai-service` | `backend` |
| `generation.dlx` | Dead-letter exchange | брокер | нет |

| Очередь | Exchange | Routing key |
|---|---|---|
| `generation.analysis.requests` | `generation.commands` | `analysis.request` |
| `generation.commit.requests` | `generation.commands` | `commit.request` |
| `generation.pull_request.requests` | `generation.commands` | `pull_request.request` |
| `generation.analysis.results` | `generation.results` | `analysis.result` |
| `generation.commit.results` | `generation.results` | `commit.result` |
| `generation.pull_request.results` | `generation.results` | `pull_request.result` |

### Параметры очередей

Очереди из таблицы выше: durable, тип quorum, с параметрами:

```json
{
    "x-queue-type": "quorum",
    "x-dead-letter-exchange": "generation.dlx",
    "x-dead-letter-routing-key": "<queue_name>.dlq",
    "x-delivery-limit": 5
}
```

Для каждой из них существует DLQ `<queue_name>.dlq` (durable, quorum, без параметров выше), привязанная к `generation.dlx` с routing key `<queue_name>.dlq`.

## Общие структуры данных

### CommitType
Тип коммита. Набор значений совпадает с полем `type` структуры `Commit` в HTTP API и определяется там.

### StructuredChanges
Результат анализа. Структура будет определена позже.

```json
json
```

### GenerationError
Описание причины `failed`. Предназначено для логов и диагностики.

```json
{
    "code": "provider_unavailable" | "invalid_provider_response" | "internal_error",
    "message": "string"
}
```

## Сообщения

### Список сообщений
[`analysis.request`](#analysisrequest)\
[`analysis.result`](#analysisresult)

[`commit.request`](#commitrequest)\
[`commit.result`](#commitresult)

[`pull_request.request`](#pull_requestrequest)\
[`pull_request.result`](#pull_requestresult)

## Analysis

### [analysis.request](#список-сообщений)
Команда на анализ изменения. Публикуется при добавлении изменения в ветку.

| | |
|---|---|
| Exchange / Routing key | `generation.commands` / `analysis.request` |
| Очередь | `generation.analysis.requests` |
| Ответ | [`analysis.result`](#analysisresult) |

#### Payload
```json
{
    "analysis_id": "UUID",
    "raw_diff": "string"
}
```

- `raw_diff` - diff целиком, непустая строка.

### [analysis.result](#список-сообщений)
Результат анализа изменения.

| | |
|---|---|
| Exchange / Routing key | `generation.results` / `analysis.result` |
| Очередь | `generation.analysis.results` |
| Ответ на | [`analysis.request`](#analysisrequest) |

#### Payload
```json
{
    "analysis_id": "UUID",
    "status": "done" | "failed",
    "structured_changes": StructuredChanges,
    "error": GenerationError
}
```

- `status: "done"` - `structured_changes` обязателен, `error` отсутствует.
- `status: "failed"` - `error` обязателен, `structured_changes` отсутствует.

## Commits

### [commit.request](#список-сообщений)
Команда на генерацию сообщения коммита. Публикуется после `analysis.result` со статусом `done` и при `POST /changes/{change_id}/commit`.

| | |
|---|---|
| Exchange / Routing key | `generation.commands` / `commit.request` |
| Очередь | `generation.commit.requests` |
| Ответ | [`commit.result`](#commitresult) |

#### Payload
```json
{
    "commit_id": "UUID",
    "structured_changes": StructuredChanges
}
```

### [commit.result](#список-сообщений)
Результат генерации сообщения коммита.

| | |
|---|---|
| Exchange / Routing key | `generation.results` / `commit.result` |
| Очередь | `generation.commit.results` |
| Ответ на | [`commit.request`](#commitrequest) |

#### Payload
```json
{
    "commit_id": "UUID",
    "status": "done" | "failed",
    "type": CommitType,
    "scope": "string" | null,
    "message": "string",
    "error": GenerationError
}
```

- `status: "done"` - обязательны `type`, `scope` (может быть `null`) и `message`, `error` отсутствует.
- `status: "failed"` - обязателен `error`, поля `type`, `scope`, `message` отсутствуют.

## Pull Requests

### [pull_request.request](#список-сообщений)
Команда на генерацию заголовка и описания PR. Публикуется при `POST /branches/{branch_id}/pull-request`.

| | |
|---|---|
| Exchange / Routing key | `generation.commands` / `pull_request.request` |
| Очередь | `generation.pull_request.requests` |
| Ответ | [`pull_request.result`](#pull_requestresult) |

#### Payload
```json
{
    "pull_request_id": "UUID",
    "branch_name": "feature/user-search",
    "changes": [
        {
            "sequence_number": 1,
            "structured_changes": StructuredChanges,
            "commit": {
                "type": CommitType,
                "scope": "string" | null,
                "message": "string"
            } | null
        }
    ]
}
```

- `changes` - все изменения ветки по возрастанию `sequence_number`, минимум одно.
- `commit` - актуальное сообщение коммита, в том числе отредактированное пользователем. `null`, если коммит не в статусе `done`.

### [pull_request.result](#список-сообщений)
Результат генерации описания PR.

| | |
|---|---|
| Exchange / Routing key | `generation.results` / `pull_request.result` |
| Очередь | `generation.pull_request.results` |
| Ответ на | [`pull_request.request`](#pull_requestrequest) |

#### Payload
```json
{
    "pull_request_id": "UUID",
    "status": "done" | "failed",
    "title": "string",
    "description": "string",
    "error": GenerationError
}
```

- `status: "done"` - обязательны `title` и `description`, `error` отсутствует.
- `status: "failed"` - обязателен `error`, поля `title` и `description` отсутствуют.

# Логистика.Про — Лабораторная №3 (REST API)

## Запуск
1. `docker compose up -d` (PostgreSQL + MinIO)
2. `alembic upgrade head`
3. `pip install -r requirements.txt`
4. `uvicorn main:app --reload`

## Таблицы БД

### users
| Поле | Тип | Описание |
|---|---|---|
| id | integer PK | Идентификатор |
| username | varchar(50) | Уникальное имя |

### transports
| Поле | Тип | Описание |
|---|---|---|
| id | integer PK | Идентификатор |
| title | varchar(100) | Название |
| description | varchar(500) | Описание |
| status | varchar(20) | draft / published / deleted |
| image_url | varchar(500) | Имя файла в MinIO (или путь заглушки) |
| video_url | varchar(500) | Имя файла в MinIO |
| speed_kmh | integer | Скорость |
| price_per_km | integer | Тариф |
| created_at | timestamp | Дата создания |
| published_at | timestamp | Дата публикации |
| author_id | integer FK users.id | Создатель |
| likes | integer | Счётчик лайков |

### likes
| Поле | Тип | Описание |
|---|---|---|
| id | integer PK | Идентификатор |
| user_id | integer FK users.id | Кто лайкнул |
| transport_id | integer FK transports.id | Что лайкнули |

Уникальное ограничение: `(user_id, transport_id)`.

## HTTP API

| Метод | URL | Описание | Тело / Параметры |
|---|---|---|---|
| GET | `/api/transports` | Список published | `?search=&tariff_from=&tariff_to=` |
| GET | `/api/feed` | Лента published | — |
| GET | `/api/feed/{id}/next` | Следующая в ленте | `?next=true` |
| GET | `/api/drafts` | Черновик текущего юзера | — |
| POST | `/api/transports` | Создать черновик + файлы | `multipart/form-data`: `title`, `image`, `video` |
| PUT | `/api/transports/{id}/publish` | Опубликовать | JSON: `description`, `speed_kmh`, `price_per_km` |
| DELETE | `/api/transports/{id}` | Soft delete | — |
| POST | `/api/transports/{id}/like` | Лайк 0/1 | JSON: `{"value": 0 \| 1}` |
| POST | `/api/users/register` | Регистрация | JSON: `{"username": "..."}` |
| POST | `/api/users/login` | Заглушка | — |
| POST | `/api/users/logout` | Заглушка | — |

## Singleton-пользователь

`core/security.py` → `get_current_user()` возвращает пользователя с
константным `id=1` (`CURRENT_USER_ID`). Используется через
`Depends(get_current_user)` во всех API-методах.
В лабораторной №4 будет заменена на JWT-аутентификацию.

## Статусы услуг

- `draft` — черновик, виден только автору через `/api/drafts`
- `published` — опубликовано, видно всем в `/api/transports` и `/api/feed`
- `deleted` — soft delete, клиенту не передаётся

Переходы: `draft → published` (PUT publish), `published → deleted` (DELETE).
Обратно в черновик вернуть нельзя.
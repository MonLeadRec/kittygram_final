# Kittygram

Kittygram — сервис для публикации котиков с фотографиями, годом рождения,
цветом и достижениями. Пользователи регистрируются, добавляют и редактируют
питомцев, а админ-зона Django позволяет управлять базой.

Проект собирается из трёх Docker-образов (backend, frontend, gateway) и
PostgreSQL. CI/CD в GitHub Actions прогоняет линтер и тесты, а при пуше в
`main` собирает образы и публикует их на Docker Hub.

## Стек технологий

- Python 3.12, Django 5.1, Django REST Framework, Djoser
- PostgreSQL 13
- React (frontend, готовый)
- Nginx 1.22 (gateway)
- Docker, Docker Compose
- GitHub Actions
- gunicorn как WSGI-сервер

## Структура

```
kittygram_final/
├── .github/workflows/main.yml     # CI/CD
├── backend/                       # Django API
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/                      # React (готовый)
│   └── Dockerfile
├── nginx/                         # Gateway
│   ├── Dockerfile
│   └── nginx.conf
├── docker-compose.yml             # Локальная сборка из исходников
├── docker-compose.production.yml  # Запуск опубликованных образов
├── .env.example
└── kittygram_workflow.yml
```

## Переменные окружения (`.env`)

Скопируйте `.env.example` в `.env` и заполните значения:

| Переменная          | Описание                                       |
|---------------------|------------------------------------------------|
| `SECRET_KEY`        | Секретный ключ Django                          |
| `DEBUG`             | `True`/`False` (по умолчанию `False`)          |
| `ALLOWED_HOSTS`     | Хосты через запятую: `localhost,127.0.0.1`     |
| `USE_SQLITE`        | `True` — SQLite, иначе PostgreSQL              |
| `POSTGRES_DB`       | Имя БД                                         |
| `POSTGRES_USER`     | Пользователь БД                                |
| `POSTGRES_PASSWORD` | Пароль БД                                      |
| `DB_HOST`           | Хост PostgreSQL (в Compose — `db`)             |
| `DB_PORT`           | Порт PostgreSQL (`5432`)                       |

## Локальный запуск из исходников

```bash
cp .env.example .env
docker compose up --build -d
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py collectstatic --noinput
docker compose exec backend cp -r /app/collected_static/. /backend_static/static/
```

Приложение откроется на <http://localhost:8000/>.

## Запуск из опубликованных образов

```bash
cp .env.example .env
docker compose -f docker-compose.production.yml pull
docker compose -f docker-compose.production.yml up -d
docker compose -f docker-compose.production.yml exec backend python manage.py migrate
docker compose -f docker-compose.production.yml exec backend python manage.py collectstatic --noinput
docker compose -f docker-compose.production.yml exec backend cp -r /app/collected_static/. /backend_static/static/
```

Приложение откроется на <http://localhost:8001/>. Taski (если запущен на
8000) продолжит работать параллельно.

Создать суперпользователя:

```bash
docker compose -f docker-compose.production.yml exec backend python manage.py createsuperuser
```

## Проверки

- Регистрация и вход на <http://localhost:8001/> работают.
- Создание и редактирование котика, загрузка картинки — работают.
- API доступно по `/api/`, админка — по `/admin/`, статика админки
  раздаётся Nginx.
- Загруженные медиа сохраняются в volume `media`; после
  `docker compose -f docker-compose.production.yml restart` данные и
  картинки остаются на месте.

## CI/CD (GitHub Actions)

Workflow хранится в `.github/workflows/main.yml` и срабатывает на пуш в
любую ветку.

1. **backend_tests** — `ruff check backend/` + `python manage.py test` с
   временным PostgreSQL из service-контейнера.
2. **frontend_tests** — `npm ci && npm run build`.
3. **build_and_push_images** — только при пуше в `main`/`master`: собирает
   и публикует образы `kittygram_backend`, `kittygram_frontend`,
   `kittygram_gateway` на Docker Hub.

В настройках репозитория GitHub → Settings → Secrets and variables →
Actions нужно добавить:

- `DOCKER_USERNAME` — логин на Docker Hub;
- `DOCKER_PASSWORD` — access token Docker Hub.

Имя пользователя из `DOCKER_USERNAME` подставляется в теги образов, чтобы
не хранить его явно в коде.

## Автор

Rodion Bykhtiarov

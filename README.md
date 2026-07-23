# Kittygram

Kittygram — социальная сеть для любителей котиков. Пользователи могут регистрироваться, добавлять своих питомцев с фото, указывать их имя, дату рождения и достижения, а также просматривать анкеты котиков других пользователей.

Проект полностью упакован в Docker-контейнеры и разворачивается на удалённом сервере автоматически через CI/CD-пайплайн на GitHub Actions: при пуше в ветку `main` код тестируется, из него собираются и публикуются на Docker Hub образы, после чего сервер обновляет и перезапускает контейнеры.

## Стек технологий

- **Backend:** Python, Django, Django REST Framework, Gunicorn
- **Frontend:** React
- **База данных:** PostgreSQL
- **Веб-сервер:** Nginx
- **Контейнеризация:** Docker, Docker Compose
- **CI/CD:** GitHub Actions
- **Реестр образов:** Docker Hub

## Архитектура

Проект состоит из четырёх контейнеров:

| Контейнер | Образ | Назначение |
|---|---|---|
| `gateway` | `username/kittygram_gateway` | Nginx: раздаёт статику и медиафайлы, проксирует запросы к backend и frontend |
| `backend` | `username/kittygram_backend` | Django-приложение, REST API |
| `frontend` | `username/kittygram_frontend` | Собранное React-приложение |
| `db` | `postgres:13` | База данных PostgreSQL |

Docker volumes:

- `static` — файлы статики backend и frontend, доступен `backend`, `frontend` и `gateway`;
- `media` — файлы, загруженные пользователями, доступен `backend` и `gateway`;
- `pg_data` — данные PostgreSQL, доступен `db`.

Nginx на сервере распределяет входящие запросы между несколькими проектами (Kittygram, Taski и др.) по доменным именам/портам.

## CI/CD

Workflow описан в `.github/workflows/main.yml` и запускается автоматически при пуше в ветку `main`. Пайплайн выполняет последовательно:

1. **tests** — проверка backend по PEP8 (flake8) и запуск тестов backend и frontend.
2. **build_and_push** — сборка Docker-образов `backend`, `frontend`, `gateway` и публикация их на Docker Hub.
3. **deploy** — подключение к серверу по SSH, копирование `docker-compose.production.yml`, обновление образов (`docker compose pull`), перезапуск контейнеров, сборка статики backend, копирование её в volume и применение миграций Django.
4. **send_message** — уведомление в Telegram об успешном завершении деплоя.

## Как развернуть проект локально

1. Клонировать репозиторий:

   ```bash
   git clone https://github.com/SleepyBeaver/kittygram_final.git
   cd kittygram_final
   ```

2. Создать файл `.env` в корне проекта на основе `.env.example` и заполнить его:

   ```env
   POSTGRES_DB=kittygram
   POSTGRES_USER=kittygram_user
   POSTGRES_PASSWORD=kittygram_password
   DB_NAME=kittygram
   DB_HOST=db
   DB_PORT=5432
   SECRET_KEY=your-django-secret-key
   ALLOWED_HOSTS=127.0.0.1,localhost
   DEBUG=False
   ```

3. Запустить контейнеры:

   ```bash
   docker compose up -d --build
   ```

4. Выполнить миграции, собрать статику и создать суперпользователя:

   ```bash
   docker compose exec backend python manage.py migrate
   docker compose exec backend python manage.py collectstatic
   docker compose exec backend cp -r /app/collected_static/. /backend_static/static/
   docker compose exec backend python manage.py createsuperuser
   ```

5. Проект будет доступен по адресу [http://localhost:9000](http://localhost:9000).

## Развёртывание на удалённом сервере

Деплой выполняется автоматически при пуше в `main`. Для его работы в настройках репозитория GitHub (Settings → Secrets and variables → Actions) должны быть добавлены секреты:

| Секрет | Описание |
|---|---|
| `DOCKER_USERNAME` / `DOCKER_PASSWORD` | Доступ к Docker Hub |
| `HOST` | IP-адрес сервера |
| `USER` | Имя пользователя на сервере |
| `SSH_KEY` / `SSH_PASSPHRASE` | Приватный SSH-ключ и его passphrase |
| `TELEGRAM_TO` / `TELEGRAM_TOKEN` | Chat ID и токен бота для уведомлений |

На сервере предварительно должна быть создана директория `kittygram/` с файлами `docker-compose.production.yml` и `.env`, а Nginx должен быть настроен на проксирование запросов в Docker (порт `9000`).

## Автор

**SleepyBeaver** — [GitHub](https://github.com/SleepyBeaver)

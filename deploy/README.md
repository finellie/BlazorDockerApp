# Деплой на Ubuntu-сервер через Dockhand

Эта папка содержит файлы для развёртывания приложения на сервере Ubuntu с помощью панели **Dockhand**.

## Состав

| Файл | Назначение |
|------|-----------|
| `docker-compose.yml` | Compose-файл для Dockhand: два сервиса (`web` + `db`), образ приложения тянется из ghcr.io |
| `.env.example` | Шаблон переменных окружения — скопируйте в `.env` и заполните |

## Как это работает

```
GitHub (push в main)
        │
        ▼
GitHub Actions  ──►  ghcr.io/finellie/blazordockerapp:latest
        │
        ▼
Ubuntu-сервер + Dockhand  ──►  docker compose pull && up -d
```

1. При push в `main` workflow `.github/workflows/docker-publish.yml` собирает образ и публикует его в GitHub Container Registry.
2. Dockhand на сервере скачивает готовый образ и запускает контейнеры. **Сборка на сервере не выполняется** — это экономит ресурсы и ускоряет деплой.

## Пошаговая инструкция

### 1. Убедитесь, что образ опубликован

После первого push в `main` проверьте вкладку **Packages** репозитория:
https://github.com/finellie/BlazorDockerApp/pkgs/container/blazordockerapp

Если пакет создан впервые, он по умолчанию **приватный**. Чтобы сервер мог его скачать, выберите один из вариантов:

- **Сделать публичным** (проще): Package settings → Change visibility → Public.
- **Оставить приватным**: в Dockhand настройте авторизацию в ghcr.io (см. шаг 3).

### 2. Подготовьте `.env` на сервере

```bash
cp .env.example .env
nano .env
```

Обязательно замените `POSTGRES_PASSWORD` на надёжный пароль:

```bash
openssl rand -base64 24
```

### 3. Настройте Dockhand

В интерфейсе Dockhand при создании стека укажите:

| Поле | Значение |
|------|----------|
| **Compose file** | содержимое `deploy/docker-compose.yml` |
| **Env file** | содержимое `deploy/.env` |

Если образ в ghcr.io приватный, добавьте в Dockhand реестр:

- **Registry URL:** `ghcr.io`
- **Username:** ваш GitHub-логин (`finellie` или `linecode-pro`)
- **Password:** Personal Access Token с областью `read:packages`
  (создать: GitHub → Settings → Developer settings → Tokens (classic) → `read:packages`)

### 4. Запустите стек

Dockhand выполнит `docker compose up -d`. Приложение будет доступно по адресу:

```
http://<IP-сервера>:9000
```

### 5. Откройте порт в firewall

```bash
sudo ufw allow 9000/tcp
sudo ufw reload
```

## Обновление приложения

```bash
# на сервере
docker compose pull
docker compose up -d
```

Либо через Dockhand — кнопка **Pull & Recreate** для стека.

## Переменные окружения

| Переменная | Обязательна | Описание | Пример |
|------------|-------------|----------|--------|
| `APP_IMAGE` | да | Полное имя образа с тегом | `ghcr.io/finellie/blazordockerapp:latest` |
| `POSTGRES_DB` | да | Имя базы данных | `blazordockerapp` |
| `POSTGRES_USER` | да | Пользователь БД | `blazorapp` |
| `POSTGRES_PASSWORD` | да | Пароль БД — **смените!** | `openssl rand -base64 24` |
| `WEB_PORT` | да | Порт приложения на хосте | `9000` |

## Важные замечания

- **Данные БД** хранятся в именованном volume `postgres-data`. Он переживает пересоздание контейнеров. Удаляется только командой `docker compose down -v`.
- **Миграции применяются автоматически** при старте приложения (`dbContext.Database.Migrate()` в `Program.cs`) — отдельный шаг не нужен.
- **Порт БД наружу не публикуется** — Postgres доступен только контейнеру `web` по внутренней сети `test-app-network`.
- **HTTPS не настроен.** Для доступа по домену с TLS добавьте reverse proxy (Caddy/Nginx + Let's Encrypt) перед приложением.
- **Смена `POSTGRES_PASSWORD` после первого запуска** не изменит пароль в существующем volume — потребуется либо `ALTER USER` вручную, либо пересоздание volume (`down -v`, данные будут потеряны).

## Отличия от корневого `docker-compose.yml`

| | Корневой (разработка) | `deploy/` (сервер) |
|---|---|---|
| Источник образа | `build:` из исходников | `image:` из ghcr.io |
| Требуется исходный код | да | нет |
| Порт БД на хост | нет | нет |
| Назначение | локальная разработка | продакшен-деплой |

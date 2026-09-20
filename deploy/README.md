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
        │  webhook (POST)
        ▼
Ubuntu-сервер + Dockhand  ──►  docker compose pull && up -d
```

1. При push в `main` workflow `.github/workflows/docker-publish.yml` собирает образ и публикует его в GitHub Container Registry.
2. После успешной публикации workflow вызывает **webhook Dockhand**, который подтягивает новый образ и пересоздаёт контейнеры.
3. Dockhand на сервере скачивает готовый образ и запускает контейнеры. **Сборка на сервере не выполняется** — это экономит ресурсы и ускоряет деплой.

## Автодеплой (webhook)

Автодеплой настраивается через секрет `DOCKHAND_WEBHOOK_URL` в GitHub Actions.

### Настройка

1. В Dockhand создайте webhook для вашего стека (раздел **Webhooks** / **Auto-deploy**) и скопируйте URL.
2. В GitHub откройте: **Settings → Secrets and variables → Actions → New repository secret**
3. Имя: `DOCKHAND_WEBHOOK_URL`, значение: скопированный URL.

### Поведение

| Ситуация | Что происходит |
|----------|----------------|
| Секрет **не задан** | Шаг деплоя выводит предупреждение и завершается успешно — сборка не ломается |
| Секрет задан, webhook ответил 2xx | Деплой запущен, в логе `::notice::` |
| Секрет задан, webhook ответил не-2xx | Шаг падает с `::error::` — вы узнаете о проблеме |
| Pull request | Шаг деплоя **не выполняется** (образ не пушится) |

> **Важно:** webhook срабатывает только при push в `main`. Для тегов `v*` образ публикуется, но деплой не запускается — это защита от случайного деплоя по тегу.

### Ручной деплой

Если автодеплой не настроен, обновляйте вручную:

```bash
docker compose pull && docker compose up -d
```

## Health-проверки

Приложение предоставляет два endpoint'а:

| Endpoint | Назначение | Проверяет | Ответ |
|----------|-----------|-----------|-------|
| `/health` | **Liveness** — процесс жив | ничего (без внешних зависимостей) | `200 Healthy` |
| `/health/ready` | **Readiness** — готов обслуживать трафик | подключение к PostgreSQL | `200 Healthy` / `503 Unhealthy` |

Оба доступны по HTTP без редиректа на HTTPS — иначе Docker healthcheck не смог бы их опросить.

Контейнер `web` использует `/health/ready` в своём healthcheck, поэтому `docker compose ps` покажет `healthy` только когда приложение **и** база готовы к работе.

> **Зачем два endpoint'а?** Если бы liveness тоже проверял БД, то при кратковременной недоступности базы оркестратор перезапускал бы контейнер приложения — хотя проблема не в нём. Liveness отвечает только за «процесс завис», readiness — за «можно слать трафик».

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
sudo ufw allow 9115/tcp
sudo ufw reload
```

## Обновление приложения

Автоматически — при push в `main` (см. раздел «Автодеплой»).

Вручную:

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
| `WEB_PORT` | да | Порт приложения на хосте | `9115` |

### Секреты GitHub Actions

| Секрет | Обязателен | Назначение |
|--------|-----------|------------|
| `DOCKHAND_WEBHOOK_URL` | нет | URL webhook Dockhand для автодеплоя. Без него деплой пропускается |

## Важные замечания

- **Данные БД** хранятся в именованном volume `postgres-data`. Он переживает пересоздание контейнеров. Удаляется только командой `docker compose down -v`.
- **Миграции применяются автоматически** при старте приложения (`dbContext.Database.Migrate()` в `Program.cs`) — отдельный шаг не нужен.
- **Порт БД наружу не публикуется** — Postgres доступен только контейнеру `web` по внутренней сети `test-app-network`.
- **HTTPS не настроен.** Для доступа по домену с TLS добавьте reverse proxy (Caddy/Nginx + Let's Encrypt) перед приложением.
- **Смена `POSTGRES_PASSWORD` после первого запуска** не изменит пароль в существующем volume — потребуется либо `ALTER USER` вручную, либо пересоздание volume (`down -v`, данные будут потеряны).
- **Порт `WEB_PORT`** должен быть свободен и не попадать в зарезервированные диапазоны ОС. На Windows диапазоны исключений меняются после перезагрузки — проверяйте `netsh interface ipv4 show excludedportrange protocol=tcp`.

## Отличия от корневого `docker-compose.yml`

| | Корневой (разработка) | `deploy/` (сервер) |
|---|---|---|
| Источник образа | `build:` из исходников | `image:` из ghcr.io |
| Требуется исходный код | да | нет |
| Порт БД на хост | нет | нет |
| Назначение | локальная разработка | продакшен-деплой |

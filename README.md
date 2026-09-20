# BlazorDockerApp

Стандартное решение **Blazor Server (.NET 10)** с авторизацией **ASP.NET Core Identity**, базой данных **PostgreSQL** и запуском в **Docker Compose** (два контейнера: Blazor + Postgres).

## Структура решения

```
NET_Docker_Test/
├── BlazorDockerApp.slnx              # Файл решения (новый формат slnx)
├── docker-compose.yml                # Два сервиса: web (Blazor) + db (Postgres) — для разработки
├── .env.example                      # Шаблон переменных окружения
├── README.md
├── .github/
│   └── workflows/
│       └── docker-publish.yml        # CI: сборка и публикация образа в ghcr.io
├── deploy/                           # Файлы для деплоя на сервер через Dockhand
│   ├── docker-compose.yml            # Compose для сервера (образ из ghcr.io)
│   ├── .env.example                  # Шаблон переменных для сервера
│   └── README.md                     # Инструкция по деплою
├── docs/
│   └── skills/
│       └── deploy-and-verify/        # Skill: как обновить репозиторий и проверить деплой
│           └── SKILL.md
└── src/
    └── BlazorDockerApp/
        ├── BlazorDockerApp.csproj    # net10.0, Npgsql.EntityFrameworkCore.PostgreSQL
        ├── Dockerfile                # Multi-stage build (sdk:10.0 → aspnet:10.0)
        ├── .dockerignore
        ├── Program.cs                # Identity + Npgsql + авто-миграции + health endpoints
        ├── appsettings.json          # Строка подключения к PostgreSQL
        ├── Data/
        │   ├── ApplicationDbContext.cs
        │   ├── ApplicationUser.cs
        │   └── Migrations/           # Миграции Identity для PostgreSQL
        ├── Components/               # Razor-компоненты, включая Account (Identity UI)
        └── wwwroot/
            ├── app.css               # Базовые стили приложения
            └── custom.css            # Кастомные стили (цвет заголовков H1)
```

## Деплой на сервер

Для развёртывания на Ubuntu-сервере через панель **Dockhand** используйте файлы из папки [`deploy/`](deploy/README.md). Образ приложения собирается в GitHub Actions и публикуется в ghcr.io — на сервере сборка не требуется.

Полная цепочка доставки:

```
git push → GitHub Actions → ghcr.io → webhook → Dockhand → Ubuntu
```

> **Важно:** в Dockhand должна быть включена опция **«Always redeploy the stack on webhook or scheduled sync, even if no git changes are detected»**. Без неё Dockhand может пропустить пересоздание контейнеров, и приложение останется на старой версии — при этом CI покажет `success`. Подробнее в [`deploy/README.md`](deploy/README.md).

### Skill для обновления и проверки

В [`docs/skills/deploy-and-verify/`](docs/skills/deploy-and-verify/SKILL.md) описан пошаговый процесс: коммит → push → отслеживание CI → проверка webhook → подтверждение, что изменение реально на сервере. Включает диагностику типовых сбоев (пропущенный деплой, ошибки авторизации webhook).

## Требования

- .NET SDK 10.0
- Docker Desktop (с Docker Compose v2+)

## Запуск в Docker

Скопируйте шаблон переменных окружения и при необходимости измените значения:

```powershell
Copy-Item .env.example .env
```

```powershell
docker compose up -d --build
```

Приложение будет доступно по адресу: **http://localhost:9115**

| Сервис | Контейнер | Порт (хост → контейнер) |
|--------|-----------|--------------------------|
| Blazor | `blazordockerapp-web` | `${WEB_PORT}` → `8080` |
| Postgres | `blazordockerapp-db` | не публикуется (только внутри сети) |

> **Примечание:** порт хоста `9115` выбран потому, что диапазоны `8063–8162` и другие зарезервированы Windows (Hyper-V/WSL). Если порт занят, измените `WEB_PORT` в `.env`.

### Переменные окружения (`.env`)

Все настройки Postgres и порт приложения вынесены в файл `.env` в корне репозитория:

| Переменная | Назначение | Значение по умолчанию |
|------------|-----------|----------------------|
| `POSTGRES_DB` | Имя базы данных | `blazordockerapp` |
| `POSTGRES_USER` | Пользователь БД | `postgres` |
| `POSTGRES_PASSWORD` | Пароль БД | `postgres` |
| `WEB_PORT` | Порт приложения на хосте | `9115` |

Файл `.env` **не коммитится** (добавлен в `.gitignore`), так как содержит пароли. В репозитории хранится только шаблон `.env.example`.

> Для продакшена обязательно смените `POSTGRES_PASSWORD` на надёжный секрет.

### Сеть

Оба контейнера подключены к одной явно объявленной bridge-сети `test-app-network`. Связь обеспечивается встроенным DNS Docker: имя сервиса `db` резолвится в IP контейнера Postgres, поэтому в строке подключения используется `Host=db`.

Postgres **не публикует порт на хост** — он доступен только контейнеру `web` по внутренней сети. Если нужен доступ к БД с хоста (pgAdmin, локальный `dotnet run`), добавьте в сервис `db`:

```yaml
    ports:
      - "5432:5432"
```

### Health-проверки

Приложение предоставляет два endpoint'а:

| Endpoint | Назначение | Проверяет |
|----------|-----------|-----------|
| `/health` | Liveness — процесс жив | ничего (без внешних зависимостей) |
| `/health/ready` | Readiness — готов обслуживать трафик | подключение к PostgreSQL |

Контейнер `web` использует `/health/ready` в своём healthcheck, поэтому `docker compose ps` покажет `healthy` только когда приложение **и** база готовы к работе.

```powershell
curl http://localhost:9115/health        # 200 Healthy
curl http://localhost:9115/health/ready  # 200 Healthy (503 если БД недоступна)
```

### Полезные команды

```powershell
docker compose ps                 # статус контейнеров
docker compose logs -f web        # логи приложения
docker compose logs -f db         # логи PostgreSQL
docker compose down               # остановить
docker compose down -v            # остановить и удалить данные БД (volume)
```

## Запуск локально (без Docker)

Требуется PostgreSQL на `localhost:5432`. Поскольку контейнер `db` не публикует порт на хост, для локального запуска добавьте маппинг `5432:5432` в сервис `db` (см. раздел «Сеть») и поднимите только БД:

```powershell
docker compose up -d db
```

```powershell
dotnet run --project src/BlazorDockerApp
```

Строка подключения по умолчанию (в `appsettings.json`):

```
Host=localhost;Port=5432;Database=blazordockerapp;Username=postgres;Password=postgres
```

## Стили

Кастомные стили приложения находятся в `src/BlazorDockerApp/wwwroot/custom.css` и подключаются в `Components/App.razor` после `app.css`:

```html
<link rel="stylesheet" href="@Assets["app.css"]" />
<link rel="stylesheet" href="@Assets["custom.css"]" />
```

Текущее содержимое `custom.css` задаёт цвет заголовков `H1`:

```css
h1 {
    color: salmon;
}
```

> Blazor добавляет к именам статических файлов отпечаток содержимого (например, `custom.kjr289qayw.css`). Это нормально — ссылка в HTML формируется автоматически через `@Assets[...]`, а при изменении файла отпечаток меняется, что сбрасывает кэш браузера.

## Авторизация (Identity)

- Используется `Microsoft.AspNetCore.Identity.EntityFrameworkCore` с `IdentityDbContext<ApplicationUser>`.
- Схема БД создаётся автоматически при старте приложения (`dbContext.Database.Migrate()` в `Program.cs`).
- Доступны страницы `/Account/Register`, `/Account/Login`, `/Account/Manage`.
- По умолчанию `SignIn.RequireConfirmedAccount = true`. Для локальной разработки без почтового сервиса можно отключить это в `Program.cs`:

```csharp
options.SignIn.RequireConfirmedAccount = false;
```

## Работа с миграциями EF Core

```powershell
# Добавить новую миграцию
dotnet ef migrations add <Name> --project src/BlazorDockerApp --output-dir Data/Migrations

# Применить миграции вручную (обычно не требуется — применяются при старте)
dotnet ef database update --project src/BlazorDockerApp
```

## Конфигурация

| Параметр | Значение по умолчанию | Где задаётся |
|----------|----------------------|--------------|
| `POSTGRES_DB` / `POSTGRES_USER` / `POSTGRES_PASSWORD` | `blazordockerapp` / `postgres` / `postgres` | `.env` |
| `WEB_PORT` | `9115` | `.env` |
| `ConnectionStrings__DefaultConnection` | `Host=db;...` | `docker-compose.yml` (собирается из переменных `.env`) |
| `ASPNETCORE_URLS` | `http://+:8080` | `Dockerfile` / `docker-compose.yml` |
| `ASPNETCORE_ENVIRONMENT` | `Production` | `docker-compose.yml` |

> Для продакшена обязательно смените `POSTGRES_PASSWORD` в `.env` на надёжный секрет и включите HTTPS.

Готово. Создал полный набор файлов для деплоя через Dockhand.

## Созданные файлы

**`docker-compose.yml`** — compose для Dockhand. Ключевое отличие от корневого: образ **тянется готовым**, а не собирается:

```
  web:
    image: ${APP_IMAGE}          # ghcr.io/finellie/blazordockerapp:latest
    # build: отсутствует — на сервере сборки нет
```

**`.env.example`** — переменные для Dockhand:

| Переменная | Значение |
| --- | --- |
| `APP_IMAGE` | `ghcr.io/finellie/blazordockerapp:latest` |
| `POSTGRES_DB` | `blazordockerapp` |
| `POSTGRES_USER` | `blazorapp` |
| `POSTGRES_PASSWORD` | `CHANGE_ME_STRONG_PASSWORD` |
| `WEB_PORT` | `9000` |

**`README.md`** — пошаговая инструкция для Ubuntu.

**`docker-publish.yml`** — CI, который собирает и публикует образ.

## Почему добавил workflow

Вы выбрали вариант «тянуть образ из ghcr.io», но **образ там ещё не публиковался** — без CI серверу было бы нечего скачивать. Workflow запускается при push в `main`, собирает образ и пушит в реестр с тегами `latest`, `main`, а для тегов `v*` — ещё и semver.

## Проверено

| Проверка | Результат |
| --- | --- |
| `docker compose config` (deploy) | переменные подставляются, образ `ghcr.io/finellie/blazordockerapp:latest` ✅ |
| `deploy/.env` в `.gitignore` | игнорируется ✅ |
| Workflow запустился | ✅ `success` за 1m12s |
| Образ в ghcr.io | опубликован ✅ |

## ⚠️ Один шаг нужно сделать вручную

Образ опубликован, но **пакет приватный** — сервер не сможет его скачать. Проверил анонимный доступ: `401 Unauthorized`.

Выберите один из вариантов:

**Вариант А — сделать публичным (проще)😗*

1. Откройте [https://github.com/orgs/finellie/packages/container/blazordockerapp/settings](https://github.com/orgs/finellie/packages/container/blazordockerapp/settings)
2. Внизу → **Change visibility** → **Public**
**Вариант Б — оставить приватным:**

1. Создайте PAT: GitHub → Settings → Developer settings → Tokens (classic) → scope `read:packages`
2. В Dockhand добавьте реестр: URL `ghcr.io`, логин — ваш GitHub-логин, пароль — этот PAT
## Как указать в Dockhand

| Поле | Значение |
| --- | --- |
| **Compose file** | содержимое `docker-compose.yml` |
| **Env file** | содержимое `deploy/.env` (скопируйте из `.env.example` и смените пароль) |

После запуска приложение будет на `http://<IP-сервера>:9000`. Не забудьте `sudo ufw allow 9000/tcp`.

## Замечания

- **Миграции применяются автоматически** при старте — отдельный шаг не нужен.
- **Смена `POSTGRES_PASSWORD` после первого запуска** не изменит пароль в существующем volume — нужен `ALTER USER` или пересоздание volume с потерей данных.
- **HTTPS не настроен** (вы выбрали доступ по IP). Для домена с TLS понадобится reverse proxy — могу добавить Caddy в compose, если понадобится.
- В логах CI есть предупреждение о deprecated Node.js 20 в actions — не влияет на работу, но стоит обновить actions до v5 при следующем изменении workflow.
Коммит `c0a3a12` запушен в `main`.

Made changes.
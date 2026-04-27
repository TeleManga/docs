## 5. Правила выгрузки

### 5.1. Среды развёртывания

| Среда | Назначение | Особенности |
|-------|-----------|-------------|
| **dev** | Локальная разработка | Запускается через `docker compose up`. MinIO вместо S3, PostgreSQL в контейнере, hot-reload фронтенда через Vite |
| **staging** | Предрелизное тестирование | Максимально приближена к production. Деплой автоматический при влитии изменений в `main` — **до** выкладки на production, в одной цепочке релиза |
| **production** | Полноценная эксплуатация | Доступна конечным пользователям через Telegram Mini App. Деплой автоматический **после** успешной выкладки на staging|

### 5.2. Стратегия ветвления

Единая интеграционная и продуктивная линия — `main`: при влитии pull request в `main` CI/CD создаёт релиз и **последовательно** выкатывает его на staging, затем на production.

```
main
 │
 ├── feat/123/manga-search
 ├── feat/456/reading-history
 ├── fix/789/auth-token
 └── ...
```

| Ветка | Назначение | Куда мёржится |
|-------|-----------|---------------|
| `main` | Единственная долгоживущая ветка; код здесь считается готовым к релизу | -- |
| `feat/*` | Новая функциональность | `main` (через pull request) |
| `fix/*` | Исправления (в том числе срочные) | `main` (через pull request) |

### 5.3. CI/CD Pipeline

Сборка и деплой автоматизированы через CI/CD (GitHub Actions).

#### 5.3.1. Pipeline для pull request'ов (в `main`)

```
Push / PR в main
    │
    ├── 1. Lint & Format Check
    │      • Backend: dotnet format --verify-no-changes
    │      • Frontend: eslint, prettier --check
    │
    ├── 2. Build
    │      • Backend: dotnet build
    │      • Frontend: npm run build
    │
    ├── 3. Test
    │      • Backend: dotnet test
    │      • Frontend: npm run test
    │
    └── 4. Docker Image Build (проверка сборки)
```

#### 5.3.2. Pipeline при влитии в `main` (релиз → staging → production)

После merge pull request в `main`: сначала создаётся релиз и публикуются образы, затем выкладка идёт **строго по порядку** — staging, после успешных проверок — production.

```
Merge в main
    │
    ├── 1. Lint + Build + Test
    │
    ├── 2. Версионирование и релиз
    │      • Определение версии v<semver> (например, по правилам репозитория или conventional commits)
    │      • Создание Git tag v<semver> и GitHub Release
    │      • Release notes из CHANGELOG или коммитов
    │
    ├── 3. Docker Build & Push
    │      • Сборка образов backend и frontend
    │      • Push в Container Registry
    │      • Тег образов: v<semver> + latest (и при необходимости main-<short-sha> для трассировки)
    │
    ├── 4. Deploy to Staging
    │      • docker compose pull && docker compose up -d
    │      • Выполнение миграций БД: dotnet ef database update
    │      • Health check: ожидание ответа 200 от GET /health
    │
    ├── 5. Создание бэкапа БД (перед production)
    │      • pg_dump перед применением миграций на production
    │
    └── 6. Deploy to Production
           • docker compose pull && docker compose up -d
           • Выполнение миграций БД: dotnet ef database update
           • Health check: ожидание ответа 200 от GET /health
```

Пока шаг 4 не завершён успешно, шаг 6 не выполняется.

### 5.4. Процедура обновления

#### 5.4.1. Штатное обновление

1. **Бэкап.** Перед деплоем CI создаёт snapshot базы данных (`pg_dump`).
2. **Миграции БД.** Выполняются до перезапуска контейнеров.
3. **Rolling update.** Docker Compose останавливает старый контейнер и запускает новый.
4. **Health check.** После старта нового контейнера CI ожидает успешного ответа `GET /health` в течение 60 секунд. Если health check не проходит -- деплой считается неуспешным.

#### 5.4.2. Откат

В случае обнаружения критических ошибок после деплоя:

1. **Откат контейнеров.** Переключение на предыдущий Docker-образ:
   ```bash
   docker compose pull   # образ с предыдущим тегом
   docker compose up -d
   ```
2. **Откат миграций** Восстановление из бэкапа:
   ```bash
   pg_restore -d telemanga backup_<timestamp>.dump
   ```

#### 5.4.3. Обновление зависимостей инфраструктуры

| Компонент | Процедура обновления |
|-----------|---------------------|
| **PostgreSQL** | Обновление минорных версий: перезапуск контейнера с новым образом. Мажорные версии: pg_dumpall -> запуск нового контейнера -> pg_restore |
| **MinIO / S3** | Обновление образа контейнера. |
| **Nginx** | Обновление образа, проверка конфигурации (`nginx -t`), перезапуск |
| **.NET Runtime** | Обновляется в Dockerfile при сборке нового образа backend |
| **Node.js** | Обновляется в Dockerfile при сборке нового образа frontend |

### 5.5. Версионирование

Проект использует **Semantic Versioning**:

```
MAJOR.MINOR.PATCH
  │     │     └── Исправления багов, не ломающие API
  │     └──────── Новая функциональность, обратно совместимая
  └────────────── Ломающие изменения API или модели данных
```

| Пример | Ситуация |
|--------|----------|
| 1.0.0 → 1.0.1 | Исправлен баг в пересчёте average_rating |
| 1.0.1 → 1.1.0 | Добавлена фильтрация каталога по статусу манги |
| 1.1.0 → 2.0.0 | Изменён формат ответа API, требуется обновление клиента |

Версия фиксируется при влитии в `main` (автоматическое создание релиза в CI) и прописывается в:
- Git tag (`v1.2.0`)
- Docker image tag (`telemanga-backend:v1.2.0`, `telemanga-frontend:v1.2.0`)
- Заголовок ответа API (`X-App-Version: 1.2.0`)
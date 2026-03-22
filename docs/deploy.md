## 5. Правила выгрузки

### 5.1. Среды развёртывания

| Среда | Назначение | Особенности |
|-------|-----------|-------------|
| **dev** | Локальная разработка | Запускается через `docker compose up`. MinIO вместо S3, PostgreSQL в контейнере, hot-reload фронтенда через Vite |
| **staging** | Предрелизное тестирование | Максимально приближена к production. Деплой автоматический при мёрже в ветку `develop` |
| **production** | Полноценная эксплуатация | Доступна конечным пользователям через Telegram Mini App. Деплой автоматический при мёрже в ветку `main` после прохождения pipeline |

### 5.2. Стратегия ветвления (Git Flow)

```
main (production)
 │
 ├── develop (staging)
 │    │
 │    ├── feat/123/manga-search
 │    ├── feat/456/reading-history
 │    └── ...
 │
 ├── hotfix/789/fix-auth
 └── release/1.2.0
```

| Ветка | Назначение | Куда мёржится |
|-------|-----------|---------------|
| `main` | Стабильный production-код | -- |
| `develop` | Интеграционная ветка для staging | `main` (через release-ветку) |
| `feat/*` | Разработка новой функциональности | `develop` |
| `hotfix/*` | Срочные исправления production | `main` + `develop` |
| `release/*` | Подготовка релиза (фиксация версии, финальные правки) | `main` + `develop` |

### 5.3. CI/CD Pipeline

Сборка и деплой автоматизированы через CI/CD (GitHub Actions).

#### 5.3.1. Pipeline для feature-веток и pull request'ов

```
Push / PR в develop
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

#### 5.3.2. Pipeline деплоя на staging

```
Merge в develop
    │
    ├── 1. Lint + Build + Test (повторно)
    │
    ├── 2. Docker Build & Push
    │      • Сборка образов backend и frontend
    │      • Push в Container Registry
    │      • Тег образа: develop-<short-sha>
    │
    └── 3. Deploy to Staging
           • docker compose pull && docker compose up -d
           • Выполнение миграций БД: dotnet ef database update
           • Health check: ожидание ответа 200 от GET /health
```

#### 5.3.3. Pipeline деплоя на production

```
Merge в main (через release-ветку или hotfix)
    │
    ├── 1. Lint + Build + Test
    │
    ├── 2. Docker Build & Push
    │      • Тег образа: v<semver> (например, v1.2.0) + latest
    │
    ├── 3. Создание бэкапа БД
    │      • pg_dump перед применением миграций
    │
    ├── 4. Deploy to Production
    │      • docker compose pull && docker compose up -d
    │      • Выполнение миграций БД: dotnet ef database update
    │      • Health check: ожидание ответа 200 от GET /health
    │
    └── 5. Создание Git Tag + GitHub Release
           • Тег: v<semver>
           • Release notes из CHANGELOG или коммитов
```

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

Версия фиксируется при создании release-ветки и прописывается в:
- Git tag (`v1.2.0`)
- Docker image tag (`telemanga-backend:v1.2.0`, `telemanga-frontend:v1.2.0`)
- Заголовок ответа API (`X-App-Version: 1.2.0`)
## 3. Техническое описание

### 3.1. Архитектура решения

Приложение построено по клиент-серверной модели. Frontend-слой представляет собой единое React SPA. В зависимости от контекста запуска (внутри Telegram или в обычном браузере), приложение динамически адаптирует свой интерфейс и способы взаимодейтсвия с сервером.

Ключевые особенности архитектуры:

- **Единный Frontend SPA**: Веб приложение развернуто на одном домене. При загрузке оно определяет среду выполнения:
    - При запуске через Telegram Mini App фронтенд инициализирует Telegram Web App SDK, получая доступ к нативным функциям мессенджера (данные пользователя, системные темы, и т.д.).
    - При открытии в обычном браузере приложение работает как стандартное SPA, используя веб-методы для авторизации и навигации.
- **Telegram Mini App** выступает в роли хоста для фронтенда. Мини приложение является способом отображения и интеграции фронтенда внутри Telegram.
- **Единый Backend API**: независимо от способа доступа, фронтенд обращается к единому API-слою.

Оба режима используют один и тот же Backend API. Архитектура решения:

![Архитектура решения](assets/diagrams/architecture/architecture.svg)

### 3.2. Стек технологий

| Компонент | Технология | Обоснование |
|-----------|-----------|-------------|
| **Frontend** | React + TypeScript | Широкая экосистема, типобезопасность, подходит для SPA |
| **Сборка фронтенда** | Vite | Быстрая сборка, поддержка HMR |
| **Стейт-менеджмент** | React Query (TanStack Query) | Кэширование серверного состояния, автоматическая инвалидация |
| **Стилизация** | CSS Modules / Tailwind CSS | Изоляция стилей, адаптивность под мобильные устройства |
| **Telegram SDK** | @telegram-apps/sdk | Официальный SDK для Telegram Mini Apps |
| **Backend** | ASP.NET Core Web API (.NET 8+) | Высокая производительность, строгая типизация, встроенная DI |
| **ORM** | Entity Framework Core | Миграции, LINQ-запросы, поддержка PostgreSQL |
| **База данных** | PostgreSQL | Надёжная СУБД, поддержка UUID, JSONB, полнотекстовый поиск |
| **Аутентификация** | JWT + bcrypt/argon2 | Регистрация по username/email/password, авторизация по username/password |
| **Хранение изображений** | S3-совместимое хранилище (MinIO) | Масштабируемое хранение файлов, CDN-совместимость |
| **Контейнеризация** | Docker + Docker Compose | Единообразное окружение для разработки и деплоя |

### 3.3. Модель данных

ER-диаграмма расположена в `docs/assets/diagrams/ERD/ERD.drawio`.

![ER-диаграмма](assets/diagrams/ERD/ERD.svg)

#### 3.3.1. Сущности

**Manga** -- основная сущность, описывающая тайтл.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| title | varchar | NOT NULL | Название |
| description | text | NOT NULL | Описание |
| cover_image_url | varchar | NOT NULL | URL обложки |
| author | varchar | NOT NULL | Автор |
| status | enum | NOT NULL | Статус публикации (ongoing, announced, completed, dropped, paused) |
| release_date | timestamptz | NOT NULL | Дата выхода |
| views_count | int | NOT NULL, DEFAULT 0 | Количество просмотров |
| average_rating | int | NOT NULL, DEFAULT 0 | Средний рейтинг. Пересчитывается на уровне приложения (Application-слой) при каждом добавлении, изменении или удалении оценки (Rating) |
| created_at | timestamptz | NOT NULL | Дата создания записи |
| updated_at | timestamptz | NOT NULL | Дата последнего обновления |

**Chapter** -- глава манги.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| manga_id | UUID | FK → Manga.id, NOT NULL | Привязка к манге |
| number | decimal | NOT NULL | Номер главы |
| title | varchar | NULLABLE | Название главы |
| volume | int | NULLABLE | Номер тома |
| pages_count | int | NOT NULL | Количество страниц |
| views_count | int | NOT NULL, DEFAULT 0 | Количество просмотров |
| created_at | timestamptz | NOT NULL | Дата создания |
| updated_at | timestamptz | NOT NULL | Дата обновления |

**Page** -- страница главы.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| chapter_id | UUID | FK → Chapter.id, NOT NULL | Привязка к главе |
| page_number | int | NOT NULL | Номер страницы |
| image_url | varchar | NOT NULL | URL изображения |

**User** -- пользователь системы.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| username | varchar | UNIQUE, NOT NULL | Имя пользователя |
| email | varchar | UNIQUE, NOT NULL | Электронная почта |
| password_hash | varchar | NOT NULL | Хеш пароля |
| avatar_url | varchar | NULLABLE | URL аватара |
| role | enum | NOT NULL, DEFAULT 'user' | Роль (user, admin) |
| created_at | timestamptz | NOT NULL | Дата регистрации |
| updated_at | timestamptz | NOT NULL | Дата обновления |

**Rating** -- оценка манги пользователем.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| user_id | UUID | FK → User.id, NOT NULL | Кто поставил |
| manga_id | UUID | FK → Manga.id, NOT NULL | Какой манге |
| rating | int | NOT NULL, CHECK (1..10) | Оценка от 1 до 10 |
| created_at | timestamptz | NOT NULL | Дата оценки |

Уникальное ограничение: `(user_id, manga_id)` -- один пользователь может оценить мангу только один раз.

**Comment** -- комментарий к манге.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| user_id | UUID | FK → User.id, NOT NULL | Автор комментария |
| manga_id | UUID | FK → Manga.id, NOT NULL | К какой манге |
| content | text | NOT NULL | Текст комментария |
| created_at | timestamptz | NOT NULL | Дата создания |
| updated_at | timestamptz | NOT NULL | Дата обновления |

**ReadingHistory** -- прогресс чтения.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| user_id | UUID | FK → User.id, NOT NULL | Пользователь |
| manga_id | UUID | FK → Manga.id, NOT NULL | Манга |
| chapter_id | UUID | FK → Chapter.id, NOT NULL | Последняя глава |
| last_page_number | int | NOT NULL | Последняя прочитанная страница |

Уникальное ограничение: `(user_id, manga_id)` -- одна запись прогресса на пару пользователь-манга.

**UserLibrary** -- библиотека пользователя.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| user_id | UUID | FK → User.id, NOT NULL | Пользователь |
| manga_id | UUID | FK → Manga.id, NOT NULL | Манга |
| status | enum | NOT NULL | Статус (reading, completed, plan_to_read, dropped, favorite) |
| added_at | timestamptz | NOT NULL | Дата добавления |

Уникальное ограничение: `(user_id, manga_id)`.

**Genre** -- жанр.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| name | varchar | UNIQUE, NOT NULL | Название жанра |
| description | text | NULLABLE | Описание жанра |

**Tag** -- тег.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | UUID | PK | Идентификатор |
| name | varchar | UNIQUE, NOT NULL | Название тега |

**manga_genre** -- связь манги и жанров (many-to-many).

| Поле | Тип | Ограничения |
|------|-----|-------------|
| manga_id | UUID | PK, FK → Manga.id |
| genre_id | UUID | PK, FK → Genre.id |

**manga_tags** -- связь манги и тегов (many-to-many).

| Поле | Тип | Ограничения |
|------|-----|-------------|
| manga_id | UUID | PK, FK → Manga.id |
| tag_id | UUID | PK, FK → Tag.id |

#### 3.3.2. Связи между сущностями

| Связь | Тип | Описание |
|-------|-----|----------|
| Manga → Chapter | 1 : N | У манги может быть много глав |
| Chapter → Page | 1 : N | У главы может быть много страниц |
| Manga ↔ Genre | M : N | Манга может иметь несколько жанров, жанр привязан к нескольким мангам (через manga_genre) |
| Manga ↔ Tag | M : N | Манга может иметь несколько тегов, тег привязан к нескольким мангам (через manga_tags) |
| User → Rating | 1 : N | Пользователь может поставить много оценок (по одной на мангу) |
| Manga → Rating | 1 : N | У манги может быть много оценок |
| User → Comment | 1 : N | Пользователь может оставить много комментариев |
| Manga → Comment | 1 : N | У манги может быть много комментариев |
| User → ReadingHistory | 1 : N | Пользователь имеет историю чтения для каждой манги |
| User → UserLibrary | 1 : N | Пользователь может добавить в библиотеку много манг |
| Manga → UserLibrary | 1 : N | Манга может быть в библиотеках многих пользователей |

### 3.4. API: группы эндпоинтов

#### 3.4.1. Аутентификация

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/api/auth/register` | Регистрация нового пользователя (username, email, password), возвращает JWT |
| POST | `/api/auth/login` | Авторизация по username + password, возвращает JWT |
| POST | `/api/auth/refresh` | Обновление JWT-токена |

#### 3.4.2. Манга

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/manga` | Список манги с пагинацией, фильтрами, сортировкой и поиском (см. ниже) | Все |
| GET | `/api/manga/{id}` | Детальная информация о манге | Все |
| POST | `/api/manga` | Создание манги | admin |
| PUT | `/api/manga/{id}` | Обновление манги | admin |
| DELETE | `/api/manga/{id}` | Удаление манги | admin |

**Query-параметры `GET /api/manga`:**

| Параметр | Тип | Описание |
|----------|-----|----------|
| `search` | string | Поиск по названию манги (полнотекстовый поиск через PostgreSQL `tsvector`) |
| `genres` | string | Фильтр по жанрам (список UUID через запятую) |
| `tags` | string | Фильтр по тегам (список UUID через запятую) |
| `status` | string | Фильтр по статусу манги (ongoing, completed и т.д.) |
| `sortBy` | string | Поле сортировки: `popularity`, `rating`, `created_at`, `title` (по умолчанию `popularity`) |
| `sortOrder` | string | Направление сортировки: `asc`, `desc` (по умолчанию `desc`) |
| `page` | int | Номер страницы (по умолчанию 1) |
| `pageSize` | int | Размер страницы (по умолчанию 20, максимум 50) |

#### 3.4.3. Главы

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/manga/{mangaId}/chapters` | Список глав манги | Все |
| GET | `/api/chapters/{id}` | Информация о главе | Все |
| POST | `/api/manga/{mangaId}/chapters` | Добавление главы | admin |
| PUT | `/api/chapters/{id}` | Обновление главы | admin |
| DELETE | `/api/chapters/{id}` | Удаление главы | admin |

#### 3.4.4. Страницы

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/chapters/{chapterId}/pages` | Список страниц главы | Все |
| POST | `/api/chapters/{chapterId}/pages` | Загрузка страниц | admin |
| DELETE | `/api/pages/{id}` | Удаление страницы | admin |

#### 3.4.5. Оценки

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| POST | `/api/manga/{mangaId}/rating` | Поставить / обновить оценку | user, admin |
| GET | `/api/manga/{mangaId}/rating` | Получить свою оценку | user, admin |
| DELETE | `/api/manga/{mangaId}/rating` | Удалить оценку | user, admin |

#### 3.4.6. Комментарии

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/manga/{mangaId}/comments` | Список комментариев (пагинация) | Все |
| POST | `/api/manga/{mangaId}/comments` | Создание комментария | user, admin |
| PUT | `/api/comments/{id}` | Редактирование своего комментария | автор / admin |
| DELETE | `/api/comments/{id}` | Удаление комментария | автор / admin |

#### 3.4.7. Библиотека пользователя

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/library` | Список манги в библиотеке (фильтр по статусу) | user, admin |
| POST | `/api/library` | Добавить мангу в библиотеку | user, admin |
| PUT | `/api/library/{id}` | Изменить статус | user, admin |
| DELETE | `/api/library/{id}` | Удалить из библиотеки | user, admin |

#### 3.4.8. История чтения

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/reading-history/{mangaId}` | Получить прогресс по манге | user, admin |
| PUT | `/api/reading-history/{mangaId}` | Обновить прогресс чтения (chapter_id, last_page_number в теле запроса) | user, admin |

#### 3.4.9. Жанры и теги

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/genres` | Список жанров | Все |
| POST | `/api/genres` | Создание жанра | admin |
| DELETE | `/api/genres/{id}` | Удаление жанра | admin |
| GET | `/api/tags` | Список тегов | Все |
| POST | `/api/tags` | Создание тега | admin |
| DELETE | `/api/tags/{id}` | Удаление тега | admin |

#### 3.4.10. Изображения

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/images/{*path}` | Получение изображения по относительному пути из S3-хранилища (прокси-контроллер, подробнее в п. 3.8) | Все |

#### 3.4.11. Системные

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/health` | Health check -- проверка работоспособности сервиса (статус БД, подключение к S3). | Без аутентификации |

### 3.5. Архитектура Backend

Backend реализуется как **модульный монолит** по принципам **Clean Architecture**.

#### 3.5.1 Слои **Clean Architecture**:

![Clean Architecture](assets/diagrams/clean-architecture/clean-architecture.svg)

| Слой | Ответственность |
|------|-----------------|
| **Presentation** | HTTP-контроллеры, маршрутизация, middleware аутентификации и авторизации, валидация входных данных |
| **Application** | Сценарии использования (use cases), координация логики (Application Services), DTO-объекты, маппинг данных |
| **Domain** | Доменные сущности, интерфейсы сервисов |
| **Infrastructure** | Реализация контрактов (EF Core: DbContext/Migrations), работа с PostgreSQL, хранилище файлов (S3 Client), реализация JWT-сервиса |

### 3.6. Архитектура Frontend

Frontend реализуется как единое SPA на React с использованием TypeScript. Приложение работает как в режиме Telegram Mini App, так и как standalone веб-приложение в браузере. Модуль `telegram/` подключается условно: при запуске внутри Telegram активируется Web App SDK (тема, навигация, haptic feedback), в браузере приложение работает автономно.

### 3.7. Аутентификация и авторизация

**Регистрация:**

1. При первом запуске приложения (как в Telegram Mini App, так и в браузере) пользователю отображается форма регистрации.
2. Пользователь заполняет поля: `username`, `email`, `password`.
3. Frontend отправляет данные на Backend (`POST /api/auth/register`).
4. Backend валидирует данные (уникальность username и email, сложность пароля), хеширует пароль (bcrypt/argon2), создаёт запись в таблице User и возвращает JWT-токен.

**Авторизация:**

1. При повторном входе (если JWT истёк) пользователю отображается форма авторизации.
2. Пользователь вводит `username` и `password`.
3. Frontend отправляет данные на Backend (`POST /api/auth/login`).
4. Backend находит пользователя по username, сверяет хеш пароля. При совпадении возвращает JWT-токен.

**Использование токена:**

1. Frontend сохраняет JWT и передаёт его в заголовке `Authorization: Bearer <token>` при каждом запросе.
2. Backend проверяет JWT и определяет роль пользователя для авторизации доступа к эндпоинтам.
3. При истечении токена Frontend запрашивает новый через `POST /api/auth/refresh`.

### 3.8. Хранение и доставка изображений

Изображения страниц манги и обложки хранятся в S3-совместимом объектном хранилище (MinIO для dev/staging, AWS S3 или аналог для production). В базе данных хранятся только относительные пути к файлам (например, `manga/{mangaId}/chapters/{chapterId}/pages/{pageNumber}.jpg`).

**Загрузка (upload):**

1. Администратор загружает изображение через `POST /api/chapters/{chapterId}/pages`.
2. Backend принимает файл, генерирует путь по конвенции `manga/{mangaId}/chapters/{chapterId}/pages/{pageNumber}.{ext}`, загружает в S3-хранилище и сохраняет относительный путь в поле `image_url` таблицы Page.

**Доставка (download):**

Запросы к изображениям проксируются через Backend-контроллер:

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/images/{*path}` | Получение изображения по относительному пути | Все |

Выбрана схема с прокси-контроллером (а не прямые ссылки на MinIO) по следующим причинам:

- **Контроль доступа.** Backend может проверять JWT и ограничивать доступ к контенту при необходимости (например, для платного контента в будущем).
- **Сокрытие инфраструктуры.** Клиент не знает о внутренней топологии хранилища; URL остаётся стабильным при миграции между провайдерами (MinIO -> S3 -> другой).
- **Единая точка входа.** Все запросы идут через один домен, что упрощает CORS-конфигурацию и TLS-терминацию.

**Оптимизация производительности:**

- Backend при проксировании использует потоковую передачу (stream), не загружая весь файл в память.
- Заголовки `Cache-Control` и `ETag` устанавливаются для кэширования на стороне клиента и промежуточных прокси.
- В production рекомендуется поставить CDN (CloudFlare / AWS CloudFront) перед Backend, который будет кэшировать ответы `/api/images/*` на edge-серверах.

### 3.9. Развёртывание

![Развёртывание](assets/diagrams/deployment/deployment.svg)

Исходный файл диаграммы: `docs/assets/diagrams/deployment/deployment.drawio`.

Для продуктивного окружения предполагается использование:

- Reverse proxy (Nginx / Traefik) с TLS-терминацией (обязательно HTTPS для Telegram Mini App).
- Managed PostgreSQL (при наличии).
- Managed S3 или CDN для изображений.

## 3. Техническое описание

### 3.1. Архитектура решения

Приложение построено по клиент-серверной модели. Frontend-слой представляет собой единое React SPA.

Ключевые особенности архитектуры:

- **Единный Frontend SPA**: Веб приложение развернуто на одном домене.
- **Telegram Mini App** является альтернативным способом отображения и интеграции фронтенда внутри Telegram.
- **Единый Backend API**: независимо от способа доступа, фронтенд обращается к единому API-слою.

Архитектура решения:

![Архитектура решения](assets/diagrams/service-diagram/service-diagram.svg)

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

1. **Comics** $-$ основная сущность, описывающая комикс.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| title | varchar | NOT NULL | Название |
| description | text | NOT NULL | Описание |
| image_url | varchar | NOT NULL | URL обложки |
| author | varchar | NOT NULL | Автор |
| country | varchar | NOT NULL | Страна |
| chapter_count | int | NOT NULL | количество глав комикса |
| status | enum | NOT NULL | Статус публикации (ongoing, announced, completed, dropped, paused) |
| kind | enum | NOT NULL | Тип (manga, manhwa, manhua) |
| release_date | timestamptz | NOT NULL | Дата выхода |
| average_rating | numeric(3,2) | NOT NULL, DEFAULT 0.00 | Средний рейтинг (0.00 - 5.00, гарантируется 2 цифры после точки и три цифры в общем). Пересчитывается на уровне приложения (Application-слой) при каждом добавлении, изменении или удалении оценки (Rating) |
| comments_count | int | NOT NULL, DEFAULT 0 | Количество комментариев |
| created_at | timestamptz | NOT NULL | Дата создания записи |
| updated_at | timestamptz | NOT NULL | Дата последнего обновления |

2. **Chapter** $-$ глава тома.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| comics_id | int | FK $\rightarrow$ Comics.id, NOT NULL | Привязка к комиксу |
| title | varchar | NOT NULL | Название главы |
| chapter_number | int | NOT NULL | Номер главы |
| pages_count | int | NOT NULL | Количество страниц в главе |

3. **Page** $-$ страница главы.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| chapter_id | int | FK $\rightarrow$ Chapter.id, NOT NULL | Привязка к главе |
| page_number | int | NOT NULL | Номер страницы |
| image_url | varchar | NOT NULL | URL на контент страницы |

4. **User** $-$ пользователь системы.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| username | varchar | UNIQUE, NOT NULL | Имя пользователя |
| email | varchar | UNIQUE, NOT NULL | Электронная почта |
| password_hash | varchar | NOT NULL | Хеш пароля |
| avatar_url | varchar | NULLABLE | URL аватара |
| role | enum | NOT NULL, DEFAULT 'user' | Роль (user, admin) |
| library_comics_count | int | NOT NULL, DEFAULT 0 | Количество комиксов добавленных в библиотеку пользователя |
| history_comics_count | int | NOT NULL, DEFAULT 0 | Количество комиксов отслеживаемых в истории чтения пользователя |
| refresh_token_hash | varchar | NOT NULL | Хеш refresh токена |
| expires_at | timestamptz | NOT NULL | Время истечения реентабельности хеша |

5. **ComicsRating** $-$ оценка комикса пользователем.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| user_id | int | FK $\rightarrow$ User.id, NOT NULL | Кто поставил |
| comics_id | int | FK $\rightarrow$ Comics.id, NOT NULL | Какому комиксу |
| rating | int | NOT NULL, CHECK (1..5) | Оценка от 1 до 5 |

Уникальное ограничение: `(user_id, comics_id)` $-$ один пользователь может оценить коимкс только один раз.

6. **Comment** $-$ комментарий к комиксу.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| user_id | int | FK $\rightarrow$ User.id, NOT NULL | Автор комментария |
| comics_id | int | FK $\rightarrow$ Comics.id, NOT NULL | К какому комиску |
| content | text | NOT NULL | Текст комментария |
| like_count | int | NOT NULL, DEFAULT 0 | Суммарное количество лайков |
| dislike_count | int | NOT NULL, DEFAULT 0 | Суммарное количество дизлайков |
| created_at | timestamptz | NOT NULL | Дата создания записи |

7. **CommentVoting** $-$ оценка комментария пользователем.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| user_id | int | FK $\rightarrow$ User.id, NOT NULL | Кто оценил |
| comment_id | int | FK $\rightarrow$ Comment.id, NOT NULL | Какой комментарий |
| voting | enum (like, dislike) | NOT NULL | Комментарий оценивается лайком/дизлайком |

Уникальное ограничение: `(user_id, comment_id)` $-$ один пользователь может оценить комментарий только один раз.

8. **ReadingHistory** $-$ история чтения.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| user_id | int | FK $\rightarrow$ User.id, NOT NULL | Пользователь |
| comics_id | int | FK $\rightarrow$ Comics.id, NOT NULL | Последний комикс |
| chapter_id | int | FK $\rightarrow$ Chapter.id, NOT NULL | Последняя глава |
| page_id | int | FK $\rightarrow$ Page.id | Последняя страница |
| created_at | timestamptz | NOT NULL | Дата создания записи |

Уникальное ограничение: `(user_id, comics_id)` $-$ одна запись прогресса на пару пользователь-комикс.

9. **Library** $-$ библиотека пользователя.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| user_id | int | FK $\rightarrow$ User.id, NOT NULL | Пользователь |
| comics_id | int | FK $\rightarrow$ Comics.id, NOT NULL | Комикс |
| status | enum (reading, completed, plan_to_read, dropped, favorite) | NOT NULL | Статус комикса, установленный пользователем |

10.   **Genre** $-$ жанр.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| name | varchar | UNIQUE, NOT NULL | Название жанра |
| description | text | NOT NULL | Описание жанра |

11.  **Tag** $-$ тег.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| name | varchar | UNIQUE, NOT NULL | Название тега |
| description | text | NOT NULL | Описание тега |

12.  **ComicsGenre** $-$ связующая таблица Comics и Genre как многие ко многим.

| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| comics_id | int | FK $\rightarrow$ Comics.id | Комикс |
| genre_id | int | FK $\rightarrow$ Genre.id | Жанр |

13. **ComicsTag** $-$ связующая таблица Comics и Tag как многие ко многим.
    
| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id | int | PK | Идентификатор |
| comics_id | int | FK $\rightarrow$ Comics.id | Комикс |
| tag_id | int | FK $\rightarrow$ Tag.id | Тег |


#### 3.3.2. Связи между сущностями

| Связь | Тип | Описание |
|-------|-----|----------|
| Comics $\rightarrow$ Chapter | 1 : N | У комикса может быть много глав |
| Comics $\rightarrow$ Rating | 1 : N | У комикса может быть много оценок |
| Comics $\rightarrow$ Comment | 1 : N | У комикса может быть много комментариев |
| Comics $\leftrightarrow$ Genre | M : N | Комиксы могут иметь несколько жанров, но как минимум 1 |
| Comics $\leftrightarrow$ Tag | M : N | Комиксы могут иметь несколько тегов, но как минимум 1 |
| Comics $\rightarrow$ Library | 1 : N | Комикс может быть в библиотеках многих пользователей |
| Chapter $\rightarrow$ Page | 1 : N | У главы может быть много страниц |
| User $\rightarrow$ ReadingHistory | 1 : N | Пользователь может читать несколько комиксов одновременно |
| Chapter $\rightarrow$ ReadingHistory | 1 : 1 | Глава сохраняется в истории пользователя для конкретного комикса (только последняя просмотренная глава) |
| Page $\rightarrow$ ReadingHistory | 1 : 1 | Страница сохраняется в истории пользователя для конкретного комикса и его конкретной главы (только последняя просмотренная страница) |
| User $\rightarrow$ ComicsRating | 1 : N | Пользователь может поставить много оценок (по одной на комикс) |
| User $\rightarrow$ CommentVoting | 1 : N | Пользователь может оценить много комментариев (но только один раз каждый) |
| User $\rightarrow$ Comment | 1 : N | Пользователь может оставить много комментариев (несколько комментариев на один комикс в том числе) |
| User $\leftrightarrow$ Comics | M : N | Пользователи могут добавлять разные комиксы в в свои библиотеки |

### 3.4. API: группы эндпоинтов

#### 3.4.1. Аутентификация

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| POST | `/api/auth/register` | Регистрация нового пользователя (username, email, password), возвращает JWT | Все |
| POST | `/api/auth/login` | Авторизация по username + password | Все |
| POST | `/api/auth/refresh` | Обновление access и refresh токенов | user, admin |
| POST | `/api/auth/logout` | Аннулирование сессии: удаление access и refresh токенов | user, admin |
| DELETE | `/api/auth/unregister` | Удаление аккаунта пользователя | user, admin |

#### 3.4.2. Комиксы

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/comics` | Список комиксов с фильтрами, сортировкой и поиском | Все |
| GET | `/api/comics/{id}` | Детальная информация о комиксе | Все |
| POST | `/api/comics` | Создание комикса | admin |
| PUT | `/api/comics/{id}` | Обновление комикса (метаданные) | admin |
| DELETE | `/api/comics/{id}` | Удаление комикса | admin |

**Query-параметры `GET /api/comics`:**

| Параметр | Тип | Описание |
|----------|-----|----------|
| `title` | string | Поиск по названию комикса |
| `author` | string | Поиск по автору комикса |
| `status` | string | Фильтр по статусу комикса (ongoing, completed и т.д.) |
| `kind` | string | Фильтр по типу комикса (manga, manhwa, manhua) |
| `release_date` | string | Фильтр по дате выпуска |
| `average_rating` | float | Фильтр по рейтингу комикса |
| `genres` | string | Фильтр по жанрам |
| `tags` | string | Фильтр по тегам |
| `sortBy` | string | Поле сортировки (по умолчанию `average_rating`) |
| `sortOrder` | string | Направление сортировки: `asc`, `desc` (по умолчанию `desc`) |
| `limit` | int | Лимит найденных комиксов |

#### 3.4.4. Главы

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/volumes/{volumeId}/chapters` | Список глав тома с фильтрами | Все |
| POST | `/api/volumes/{volumeId}/chapters` | Добавление главы | admin |
| GET | `/api/chapters/{id}` | Информация о главе | Все |
| PUT | `/api/chapters/{id}` | Обновление главы | admin |
| DELETE | `/api/chapters/{id}` | Удаление главы | admin |

**Query-параметры `GET /api/volumes/{volumeId}/chapters`:**

| Параметр | Тип | Описание |
|----------|-----|----------|
| `title` | string | Фильтр по названию главы |
| `chapter_nomer` | int | Фильтр по номеру главы |
| `limit` | int | Лимит найденных глав |

#### 3.4.5. Страницы

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/chapters/{chapterId}/pages` | Список страниц главы | Все |
| POST | `/api/chapters/{chapterId}/pages` | Добавление страницы | admin |
| GET | `/api/pages/{id}` | Информация о странице | Все |
| PUT | `/api/pages/{id}` | Обновление страницы | admin |
| DELETE | `/api/pages/{id}` | Удаление страницы | admin |

#### 3.4.6. Оценки комиксов

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| POST | `/api/comics/{comicsId}/rating` | Поставить / обновить оценку | Все |
| GET | `/api/comics/{comicsId}/rating` | Получить свою оценку | Все |
| DELETE | `/api/comics/{comicsId}/rating` | Удалить оценку | Все |

#### 3.4.7. Комментарии

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/api/comics/{comicsId}/comments` | Список комментариев | Все |
| POST | `/api/comics/{comicsId}/comments` | Создание комментария | Все |
| DELETE | `/api/comments/{id}` | Удаление комментария | Все |

#### 3.4.8 Оценки комментариев

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| POST | `/api/comments/{commentId}/vote` | Оценить комментарий | Все |
| GET | `/api/comments/{commentId}/vote` | Получить оценку комментария (свою если была) | Все |
| PUT | `/api/comments/{commentId}/vote` | Изменить оценку комментария | Все |
| DELETE | `/api/comments/{commentId}/vote` | Удалить свою оценку комментария | Все |

#### 3.4.9. Сохраненные комиксы в библиотеке пользователя

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| POST | `/api/libraries` | Добавить комикс в библиотеку | User |
| GET | `/api/libraries` | Список комиксов в библиотеке с фильтрацией | User |
| PUT | `/api/libraries/{libraryId}` | Изменить статус комикса в библиотеке | User |
| DELETE | `/api/libraries/{libraryId}` | Удалить комикс из библиотеки | User |

**Query-параметры `GET /api/libraries`:**

| Параметр | Тип | Описание |
|----------|-----|----------|
| `status` | string | Фильтр по статусу коммикса |
| `sortBy` | string | Поле сортировки (по умолчанию `status`) |
| `sortOrder` | string | Направление сортировки: `asc`, `desc` (по умолчанию `desc`) |
| `limit` | int | Лимит найденных комиксов |

Каждый из этих методов подразумевает обращение к персональной
библиотеке пользователя, id которого определяется
через JWT.

#### 3.4.10. История чтения

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| POST | `/api/reading-history` | Создать прогресс чтения по комиксу | Все |
| GET | `/api/reading-history` | Список прогрессов чтения по комиксам | Все |
| GET | `/api/reading-history/{id}` | Получить прогресс чтения по комиксу | Все |
| PUT | `/api/reading-history/{id}` | Обновить прогресс чтения | Все |
| DELETE | `/api/reading-history/{id}` | Удалить прогресс чтения | Все |

#### 3.4.11. Жанры

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| POST | `/api/genres` | Создание жанра | admin |
| GET | `/api/genres` | Список жанров | Все |
| PUT | `/api/genres/{id}` | Изменение жанра | admin |
| DELETE | `/api/genres/{id}` | Удаление жанра | admin |

#### 3.4.12. Теги

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| POST | `/api/tags` | Создание тега | admin |
| GET | `/api/tags` | Список тегов | Все |
| PUT | `/api/tags/{id}` | Изменение тега | admin |
| DELETE | `/api/tags/{id}` | Удаление тега | admin |

#### 3.4.13. Системные

| Метод | Путь | Описание | Доступ |
|-------|------|----------|--------|
| GET | `/health` | Health check $-$ проверка работоспособности сервиса (статус БД, подключение к S3). | Без аутентификации |

### 3.5. Архитектура Backend

Backend реализуется как **модульный монолит** по принципам **Clean Architecture**.

#### 3.5.1 Слои **Clean Architecture**:

![Clean Architecture](assets/diagrams/clean-architecture/clean-architecture.svg)

| Слой | Ответственность |
|------|-----------------|
| **Presentation** | HTTP-контроллеры, маршрутизация, middleware аутентификации и авторизации, валидация входных данных, DTO-объекты |
| **Application** | Сценарии использования (use cases), координация логики (Application Services) |
| **Domain** | Доменные сущности, интерфейсы сервисов |
| **Infrastructure** | Реализация контрактов (Database Context), хранилище файлов (S3 Client) |

### 3.6. Архитектура Frontend

Frontend реализуется как единое SPA на React с использованием TypeScript.

### 3.7. Аутентификация и авторизация

**Регистрация:**

1. При первом запуске приложения (как в Telegram Mini App, так и в браузере) пользователю отображается форма регистрации.
2. Пользователь заполняет поля: `username`, `email`, `password`.
3. Frontend отправляет данные на Backend (`POST /api/auth/register`).
4. Backend валидирует данные (уникальность username и email, сложность пароля), хеширует пароль (bcrypt/argon2), создаёт запись в таблице User и возвращает JWT-токен (**access** - кратковременный для предоставления прав доступа Backend'у, храниться открыто) и еще один JWT-токен (**refresh** - долговременный для обновления кратковременного, должен быть защищен и скрыт на стороне клиента, он будет использоваться в момент, когда кратковременный токен истек, данный токен должен храниться в постоянной куке). При этом в самом Backend'е храниться хешированная копия **refresh** токена для проверки с переданным пользователю.

**Авторизация:**

1. При повторном входе (если **refresh** токен истек) пользователю отображается форма авторизации.
2. Пользователь вводит `username` и `password`.
3. Frontend отправляет данные на Backend (`POST /api/auth/login`).
4. Backend находит пользователя. При успешной валидации обновляет **refresh** токен и возвращает два новых: **access** и **refresh**.

Если **refresh** токен не истек, то шаги с вводом аутентификационных данных пропускаются.

**Использование токена access:**

1. Frontend сохраняет JWT и передаёт его в заголовке `Authorization: Bearer <token>` при каждом запросе.
2. Backend проверяет JWT и определяет роль пользователя для авторизации доступа к эндпоинтам.

### 3.8. Хранение и доставка изображений

Изображения аватара, страниц комиксов и обложки хранятся в S3-совместимом объектном хранилище (MinIO). В базе данных хранятся только url на файлы.

### 3.9. Развёртывание

![Развёртывание](assets/diagrams/deployment/deployment.svg)

Для продуктивного окружения предполагается использование:

- PostgreSQL.
- S3 для изображений.

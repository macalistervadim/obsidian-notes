#  О чем эта подглава
В этой части мы настроим высокопроизводительное неблокирующее подключение к PostgreSQL с использованием драйвера `asyncpg` и современной объектно-реляционной модели SQLAlchemy 2.0, а также интегрируем управление сессиями в жизненный цикл приложения FastAPI.

* **Что сделаем:** установим библиотеки `asyncpg` и `sqlalchemy`, опишем валидируемую конфигурацию через `pydantic-settings`, инициализируем асинхронный движок `AsyncEngine`, настроим параметры пула соединений (`pool_size`, `max_overflow`), создадим фабрику сессий `async_sessionmaker` и организуем предоставление сессий в эндпоинты через FastAPI Dependency Injection (`Depends`).
* **Исходное состояние проекта:** в `docker-compose.yml` настроен и запущен контейнер с PostgreSQL 17 на порту 5432, переменные подключения зафиксированы в `.env` (по итогам подглавы 2.2).
* **Ветка в репозитории:** `chapter-02-db-connection` (готовый результат для тех, кто читает главу изолированно).

> **Что почитать для углубления:**
> * Документация SQLAlchemy 2.0: разделы *«Asynchronous I/O (asyncio)»* и *«Engine and Connection Pooling»*.
> * Документация `pydantic-settings`: управление конфигурацией из переменных окружения и `.env`-файлов.
> * Билл Любанович, *«FastAPI»* (O'Reilly, 2023) — паттерн генератора сессий баз данных через `yield` в зависимостях.
> * Тим Питерс, *«Программирование бэкенда на Python. Практическое руководство»* (БХВ, 2025) — нюансы тюнинга пула соединений под микросервисные нагрузки.

---

### Архитектура асинхронного взаимодействия с СУБД

Традиционные синхронные драйверы (например, `psycopg2`) при выполнении любого SQL-запроса блокируют поток операционной системы до тех пор, пока сервер PostgreSQL не вернет ответ по сети. В асинхронных веб-фреймворках вроде FastAPI это приводит к блокировке главного цикла событий (*Event Loop*): пока один запрос ждет завершения сложной выборки из базы, сервер не способен принимать новые подключения от других клиентов.

Для платформы `TradePulse` критически важно сохранять неблокирующий ввод-вывод:
1. **Драйвер `asyncpg`:** компилирует протокол взаимодействия с PostgreSQL на уровне байтовых потоков в корутины Python без использования C-библиотеки `libpq`.
2. **Диалект `postgresql+asyncpg`:** сообщает SQLAlchemy о необходимости маршрутизировать все операции через асинхронный интерфейс `AsyncEngine`.
3. **Пул соединений (`Connection Pool`):** открытие нового TCP-сокета и аутентификация в PostgreSQL — дорогая операция (от 10 до 50 мс). Пул держит готовые открытые коннекты в памяти и мгновенно выдает их входящим запросам, возвращая обратно после фиксации транзакции.

---

### Шаг 1. Установка зависимостей

Добавим необходимые библиотеки в основную группу проекта с помощью `uv`:

```bash
uv add "sqlalchemy[asyncio]>=2.0.35" "asyncpg>=0.29.0" "pydantic-settings>=2.5.0"
```

* `sqlalchemy[asyncio]` — ядро ORM с поддержкой неблокирующих сессий.
* `asyncpg` — быстрый асинхронный сетевой драйвер.
* `pydantic-settings` — строгая типизация и валидация конфигурации сервиса из окружения.

---

### Шаг 2. Строгая конфигурация подключения в core/config.py

Конфигурация не должна считываться через ненадежные вызовы `os.getenv()`. Воспользуемся `pydantic-settings` для автоматической валидации типов и схемы URL.

Откройте `src/tradepulse/core/config.py` и добавьте описание настроек базы данных:

```python
from pydantic import PostgresDsn, computed_field
from pydantic_core import MultiHostUrl
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    # Базовые параметры окружения
    ENVIRONMENT: str = "development"
    DEBUG: bool = False

    # Параметры подключения к PostgreSQL
    POSTGRES_USER: str = "tradepulse"
    POSTGRES_PASSWORD: str = "tradepulse_password"
    POSTGRES_HOST: str = "localhost"
    POSTGRES_PORT: int = 5432
    POSTGRES_DB: str = "tradepulse_db"
# Параметры пула соединений
    DB_POOL_SIZE: int = 10
    DB_MAX_OVERFLOW: int = 20
    DB_POOL_TIMEOUT: int = 30
    DB_ECHO: bool = False

    @computed_field  # type: ignore[prop-decorator]
    @property
    def database_url(self) -> str:
        """Динамически формирует валидную строку подключения postgresql+asyncpg."""
        return str(
            MultiHostUrl.build(
                scheme="postgresql+asyncpg",
                username=self.POSTGRES_USER,
                password=self.POSTGRES_PASSWORD,
                host=self.POSTGRES_HOST,
                port=self.POSTGRES_PORT,
                path=self.POSTGRES_DB,
            )
        )

settings = Settings()
```

---

### Шаг 3. Инициализация движка и фабрики сессий

Создадим модуль управления соединениями `src/tradepulse/core/database.py`:

```bash
touch src/tradepulse/core/database.py
```

Откройте созданный файл и настройте `AsyncEngine`, декларативную базу моделей и генератор сессий:

```python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase

from tradepulse.core.config import settings

# 1. Создание асинхронного движка с настройкой пула
engine: AsyncEngine = create_async_engine(
    settings.database_url,
    echo=settings.DB_ECHO,
    pool_size=settings.DB_POOL_SIZE,
    max_overflow=settings.DB_MAX_OVERFLOW,
    pool_timeout=settings.DB_POOL_TIMEOUT,
    pool_pre_ping=True,  # Автоматическая проверка живого сокета перед выдачей из пула
)

# 2. Фабрика создания асинхронных сессий
async_session_maker = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,  # Предотвращает перезагрузку атрибутов после commit
    autoflush=False,
)

# 3. Базовый декларативный класс для будущих ORM-моделей
class Base(DeclarativeBase):
    pass

# 4. FastAPI Dependency для безопасной выдачи сессий в эндпоинты
async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    """Генератор сессии БД: гарантирует возврат коннекта в пул и откат при ошибке."""
    async with async_session_maker() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

---

### Архитектурный разбор ключевых параметров

1. **`pool_pre_ping=True`:**  
   Критически важный флаг для микросервисов. Если межсетевой экран или сервер PostgreSQL разорвал неактивный сокет, SQLAlchemy выполнит скрытый тестовый пинг (`SELECT 1`) перед передачей соединения обработчику запроса. Если сокет мертв, он прозрачно переподключится, исключая ошибку `ConnectionResetError`.
2. **`expire_on_commit=False`:**  
   По умолчанию SQLAlchemy при вызове `commit()` инвалидирует атрибуты загруженных объектов, и при следующей попытке прочитать поле (например, `order.status`) пытается выполнить повторный синхронный запрос к базе. В асинхронном режиме это вызывает ошибку `MissingGreenlet`. Флаг `expire_on_commit=False` сохраняет загруженные данные в оперативной памяти объекта.
3. **Безопасный контекст сессии (`get_db_session`):**  
   Конструкция `try / except / finally` внутри асинхронного генератора гарантирует, что даже если в контроллере FastAPI произойдет необработанное исключение, незафиксированные данные будут сброшены (`rollback()`), а само соединение вернется в пул свободных сокетов, предотвращая утечку коннектов (*connection leak*).

---

### Шаг 4. Проверка подключения в main.py и проверка здоровья

Интегрируем проверку подключения к PostgreSQL в жизненный цикл приложения (`lifespan`) и расширим эндпоинт проверки здоровья `/health`.

Откройте `src/tradepulse/main.py` и обновите его:

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import Depends, FastAPI
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from tradepulse.core.database import engine, get_db_session
from tradepulse.core.logger import log
ger, setup_logging

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    setup_logging(env="dev")
    logger.info("Инициализация сервиса TradePulse", version="0.1.0")

    # Проверка доступности базы данных при старте
    try:
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        logger.info("Успешное подключение к пулу PostgreSQL")
    except Exception as exc:
        logger.error("Не удалось подключиться к PostgreSQL", error=str(exc))
        raise

    yield

    # Корректное закрытие соединений при остановке
    await engine.dispose()
    logger.info("Пул соединений с PostgreSQL закрыт")

app = FastAPI(
    title="TradePulse API",
    lifespan=lifespan,
)

@app.get("/health")
async def healthcheck(
    db: AsyncSession = Depends(get_db_session),
) -> dict[str, str]:
    # Выполняем проверочный неблокирующий SQL-запрос через выданную сессию
    result = await db.execute(text("SELECT version();"))
    db_version = result.scalar() or "unknown"
    
    logger.debug("Проверка здоровья сервиса и БД пройдена")
    return {
        "status": "ok",
        "database": "connected",
        "postgres_version": str(db_version),
    }
```

---

### Шаг 5. Практический запуск и валидация

Убедитесь, что контейнер с PostgreSQL запущен:

```bash
docker compose --profile dev up postgres -d
```

Запустите локальный сервер приложения:

```bash
just run
```

В выводе терминала `structlog` зафиксирует успешное подключение к базе данных:

```text
2026-10-28T06:00:01.123456Z [info     ] Инициализация сервиса TradePulse version=0.1.0
2026-10-28T06:00:01.456789Z [info     ] Успешное подключение к пулу PostgreSQL
INFO:     Application startup complete.
```

Отправьте проверочный запрос к эндпоинту `/health`:

```bash
curl -i http://127.0.0.1:8000/health
```

Ответ сервера:
```http
HTTP/1.1 200 OK
content-type: application/json

{
  "status": "ok",
  "database": "connected",
  "postgres_version": "PostgreSQL 17.0 on x86_64-pc-linux-musl..."
}
```

Выполним проверку соответствия правилам типизации и форматирования:

```bash
just check
```

Все проверки `ruff` и `mypy` должны завершиться без единой ошибки.

> **Грабли: Ошибка ConnectionRefusedError при старте**  
> Если приложение падает с ошибкой:  
> `socket.gaierror / ConnectionRefusedError: [Errno 111] Connect call failed ('127.0.0.1', 5432)`  
> убедитесь, что в файле `.env` параметр `POSTGRES_HOST` установлен как `localhost` (для локального запуска через `uv run`), а не `postgres` (имя сервиса внутри сети Docker).

---

### Контрольная точка
К этому моменту подсистема работы с реляционной СУБД настроена:
1. В проект интегрированы библиотеки `asyncpg`, `SQLAlchemy 2.0` и `pydantic-settings`.
2. В модуле `src/tradepulse/core/database.py` сконфигурирован неблокирующий пул соединений и фабрика сессий `async_sessionmaker`.
3. Реализована инъекция зависимости `get_db_session` через механизм `FastAPI.Depends()`.
4. Точка входа `main.py` проверяет доступность PostgreSQL при старте и корректно освобождает ресурсы пула при выключении.

Рабочий пример кода этого этапа доступен в ветке:  
https://github.com/macalistervadim/tradepulse/tree/chapter-02-db-connection
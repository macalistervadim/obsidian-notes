## 1.10. Статическая верификация типов: интеграция и строгая настройка mypy

### О чем эта подглава
В этой части мы подключим статический анализатор типов `mypy`, переведем проверку типов в строгий режим (*strict mode*), столкнемся с первыми реальными ошибками строгой типизации в модуле `logger.py` и разберем архитектурные способы их устранения.

* **Что сделаем:** разберем роль статического анализа в финтех-сервисах, добавим `mypy` в dev-зависимости, сконфигурируем строгий профиль проверок в `pyproject.toml`, выполним диагностический запуск, разберем конфликт вывода типов (`[assignment]`) при полиморфных объектах в `logger.py` и приведем проект к статусу `Success`.
* **Исходное состояние проекта:** в проекте настроен линтер и автоформаттер `ruff`, код модулей приведен к современным стандартам синтаксиса Python 3.13 (по итогам подглавы 1.9).
* **Ветка в репозитории:** `chapter-01-mypy-setup` (готовый результат для тех, кто читает главу изолированно).

> **Что почитать для углубления:**
> * Официальная документация Mypy: разделы *«Type inference and type annotations»* и *«Configuring mypy with pyproject.toml»*.
> * Тим Питерс, *«Программирование бэкенда на Python. Практическое руководство»* (БХВ, 2025) — раздел о статической верификации и контрактах типизации в микросервисах.
> * Билл Любанович, *«FastAPI»* (O'Reilly, 2023) — глава о взаимодействии аннотаций типов Python и моделей Pydantic.
> * PEP 484 — Type Hints (базовая спецификация системы аннотаций типов Python).

---

### Зачем строго типизировать динамический Python

Python остается языком с динамической типизацией, однако в крупных распределенных сервисах доверие проверкам типов только во время исполнения (*runtime*) несет высокие инженерные риски. В финтех-приложении уровня `TradePulse` ошибка передачи строкового тикера вместо объекта котировки или пропуск значения `None` в расчетах баланса портфеля приводит к падению воркера прямо посреди торговой сессии.

Инструмент **`mypy`** выполняет проверку кодовой базы без фактического запуска программы:
* Находит несоответствия сигнатур функций и передаваемых аргументов до попадания кода на тестовые стенды.
* Принуждает разработчика явно обрабатывать опциональные значения (`T | None`), исключая ошибку `AttributeError: 'NoneType' object has no attribute ...`.
* Валидирует контракты между слоями, гарантируя, что изменения в слое хранения (`repositories`) не сломают бизнес-логику (`services`).

---

### Шаг 1. Добавление зависимости

Как и линтер, инструмент проверки типов нужен только разработчикам и в пайплайнах CI/CD. Добавим `mypy` в группу `dev` через пакетный менеджер `uv`:

```bash
uv add --dev mypy
```

`uv` обновит манифест `pyproject.toml` и зафиксирует точные хэши в `uv.lock`.

---

### Шаг 2. Конфигурация Mypy в pyproject.toml

По умолчанию `mypy` работает в мягком режиме и пропускает нетипизированный код. Для промышленного сервиса необходим строгий профиль, запрещающий неявные типы `Any` и пропущенные сигнатуры.

Откройте `pyproject.toml` и добавьте блок конфигурации `[tool.mypy]`:

```toml
[tool.mypy]
# Версия синтаксиса Python для анализа
python_version = "3.13"

# Базовый каталог с исходным кодом (src-layout)
mypy_path = "src"

# Включение строгого режима проверок
strict = true

# Запрет неявного вывода Any при отсутствии аннотаций
disallow_untyped_defs = true
disallow_incomplete_defs = true

# Предупреждение о неиспользуемых директивах игнорирования (# type: ignore)
warn_unused_ignores = true

# Предупреждение о возврате значения Any из типизированной функции
warn_return_any = true

# Требование явной проверки на None
no_implicit_optional = true

# Детализация сообщений об ошибках
show_error_codes = true
pretty = true

# Игнорирование отсутствия stubs-файлов для внешних библиотек
[[tool.mypy.overrides]]
module = [
    "structlog.*",
]
ignore_missing_imports = true
```

---

### Шаг 3. Первый запуск анализатора и диагностика ошибок

Запустим верификацию исходного кода:

```bash
uv run mypy src
```

Терминал вернет диагностический отчет с двумя ошибками типизации в модуле логирования:

```text
src/tradepulse/core/logger.py:22: error: Incompatible types in assignment (expression has type "KeyValueRenderer", variable has type "ConsoleRenderer")  [assignment]
            renderer = structlog.processors.KeyValueRenderer()
                       ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
src/tradepulse/core/logger.py:26: error: Incompatible types in assignment (expression has type "JSONRenderer", variable has type "ConsoleRenderer")  [assignment]
            renderer = structlog.processors.JSONRenderer()
                       ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Found 2 errors in 1 file (checked 7 source files)
```

---

### Шаг 4. Анатомия ошибки: почему mypy запрещает переприсваивание

Код ошибки: `[assignment]` (несовместимые типы при повторном присваивании).

Разберем, как мыслит анализатор:
1. В блоке `if env == "dev":` мы впервые объявили локальную переменную:  
   `renderer = structlog.dev.ConsoleRenderer(colors=True)`
2. Поскольку явная аннотация типа отсутствовала, `mypy` вывел строгий тип переменной по первому присвоенному значению:  
   `renderer: ConsoleRenderer`
3. Далее в ветках `elif` и `else` мы присваиваем переменной `renderer` экземпляры классов `KeyValueRenderer` и `JSONRenderer`.
4. С точки зрения системы статических типов Python эти три класса не наследуются друг от друга напрямую. Попытка записать `KeyValueRenderer` в переменную типа `ConsoleRenderer` для `mypy` равносильна попытке записать строку в переменную, объявленную как число.

В библиотеке `structlog` все процессоры вывода реализуют протокол `structlog.types.Processor` (функция или объект с методом `__call__`). Для решения проблемы достаточно **явно объявить допустимый тип переменной перед ветвлением**.

---

### Шаг 5. Исправление модуля logger.py

Откройте файл `src/tradepulse/core/logger.py` и импортируйте контракт `Processor` из `structlog.types`.

Обновите реализацию функции `setup_logging`:

```python
import logging
import sys
from typing import Literal

import structlog
from structlog.types import Processor

EnvironmentType = Literal["dev", "test", "prod"]


def setup_logging(env: EnvironmentType = "dev") -> None:
    """Конфигурирует форматирование и уровень детализации логирования под конкретное окружение."""

    # Явно указываем общий тип процессора перед ветвлением
    renderer: Processor

    if env == "dev":
        log_level = logging.DEBUG
        renderer = structlog.dev.ConsoleRenderer(colors=True)
    elif env == "test":
        log_level = logging.WARNING
        renderer = structlog.processors.KeyValueRenderer()
    else:  # prod
        log_level = logging.INFO
        renderer = structlog.processors.JSONRenderer()

    # Конфигурация конвейера structlog
    structlog.configure(
        processors=[
            structlog.stdlib.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            renderer,
        ],
        logger_factory=structlog.PrintLoggerFactory(file=sys.stdout),
    )

    logging.basicConfig(level=log_level)


logger = structlog.get_logger()
```

---

### Шаг 6. Повторная проверка

Запустим валидацию типов снова:

```bash
uv run mypy src
```

Результат:
```text
Success: no issues found in 7 source files
```

Все файлы сервиса успешно прошли строгую верификацию.

> **Грабли: Библиотеки без встроенных типов (type stubs)**  
> Некоторые сторонние пакеты не содержат файла-маркера `py.typed` или пакетов типизации (stubs). При их импорте `mypy` прерывает проверку с ошибкой:  
> `Cannot find implementation or library stub for module named 'some_lib'`  
> 
> Правильное решение — не отключать строгий режим глобально, а изолированно объявить секцию переопределения (`overrides`) в `pyproject.toml` для конкретного проблемного пакета:
> ```toml
> [[tool.mypy.overrides]]
> module = "some_lib.*"
> ignore_missing_imports = true
> ```

---

### Контрольная точка
К этому моменту статическая типизация проекта выведена на промышленный уровень:
1. Пакет `mypy` установлен в секцию `dev`-зависимостей.
2. В файле `pyproject.toml` активирован строгий режим (`strict = true`) с поддержкой `src-layout`.
3. Разобран механизм вывода типов локальных переменных и устранен конфликт полиморфизма процессоров в `src/tradepulse/core/logger.py`.
4. Кодовая база сервиса `TradePulse` полностью валидна с точки зрения статического анализатора типов.

Рабочий пример кода этого этапа доступен в ветке:  
https://github.com/macalistervadim/tradepulse/pull/new/chapter-01-mypy-setup
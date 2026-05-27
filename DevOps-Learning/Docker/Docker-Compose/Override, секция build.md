## Расширенная конфигурация Docker Compose: секция `build` и несколько YAML-файлов

В базовых примерах Docker Compose мы, как правило, используем один файл `docker-compose.yml` и указываем готовые образы в виде `image:...`. Но на практике часто требуется:

- **Собирать Docker-образы на лету** (с помощью `build:`) напрямую из Docker Compose, без отдельного вызова `docker build`.
- **Разделить конфигурацию** на базовый и локальный (override) файлы: чтобы по-разному настраивать проект для разработки и продакшена.

В этом уроке мы разберём, как Compose поддерживает оба механизма.

### 1. Секция `build` в Docker Compose

Не всегда у вас есть готовый образ. Часто удобнее собирать образ прямо при запуске Docker Compose. Секция `build` позволяет это сделать, убирая необходимость в отдельных командах `docker build`.

#### 1.1 Базовый пример

```yaml
version: "3.8"
services:
  api:
    build: .
    ports:
      - "5000:5000"
```

- `build: .` говорит: «Возьми Dockerfile из текущей директории (`.`) и собери образ.»
- Когда делаете `docker-compose up`, Compose сначала выполнит сборку, потом запустит контейнер на базе полученного образа.

#### 1.2 Кастомизация секции `build`

```yaml
version: "3.8"
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
      args:
        ENVIRONMENT: "production"
    image: myorg/app:latest
    ports:
      - "8080:8080"
```

- **context:** директория, передаваемая в качестве контекста (аналог `docker build ./app`).
- **dockerfile:** имя файла Dockerfile (если не `Dockerfile` по умолчанию).
- **args:** build-аргументы (через `ARG` внутри Dockerfile). Здесь `ENVIRONMENT=production`.
- **image:** имя образа (например, `myorg/app:latest`). Полезно, если вы планируете потом `docker push`.

#### 1.3 Польза для CI/CD

- Можно хранить один или несколько Compose-файлов в репозитории, а в CI просто запускать `docker-compose build`.  
    Нет лишних скриптов для `docker build` и `docker run`.

### 2. `docker-compose.override.yml` и несколько YAML-файлов

По умолчанию, если рядом с `docker-compose.yml` лежит `docker-compose.override.yml`, Compose автоматически сливает их в единое целое при запуске. Настройки из `override` перекрывают параметры в основном файле.

#### 2.1 Зачем нужен `docker-compose.override.yml`?

- **Разделение окружений:** базовый файл описывает общую схему (образы, сети, тома), а override содержит локальные настройки (bind mounts, переменные окружения для разработки, debug-настройки).
- **Локальные правки:** вы можете монтировать исходники или добавлять временные переменные, не меняя основной `docker-compose.yml`.
- **Удобство переключения:** при `docker-compose up` (или `docker compose up`) и наличии override-файла всё автоматически объединится. Не нужен ручной флаг, если файл назван именно `docker-compose.override.yml`.

#### 2.2 Пример

**docker-compose.yml (базовый)**:

```yaml
version: "3.8"
services:
  web:
    image: myorg/webapp:1.0
    container_name: webapp
    ports:
      - "8080:80"
```

**docker-compose.override.yml**:

```yaml
version: "3.8"
services:
  web:
    # Вместо готового образа — build-секция (например, для локальной разработки)
    build:
      context: .
      dockerfile: Dockerfile.dev

    # Монтируем исходный код
    volumes:
      - ./:/usr/src/app

    environment:
      - APP_ENV=development
```

- Теперь, если просто ввести `docker-compose up`, Compose увидит override-файл и заменит `image: myorg/webapp:1.0` на `build: ...`, а также добавит `volumes` и `environment` переменные.
- Результат: локальная сборка + горящий код (bind mount), переменная `APP_ENV=development` и т. д.

#### 2.3 Явное перечисление нескольких файлов

```cpp
docker-compose -f docker-compose.yml -f docker-compose.override.yml up
```

Compose поочерёдно прочитает сначала `docker-compose.yml`, потом `docker-compose.override.yml`. Если вы укажите другие файлы (например, `dev.yml`, `test.yml`), то так же объединит их.

#### 2.4 Практика хранения override-файла

- **В репозитории:** если все разработчики используют одинаковые dev-настройки, имеет смысл хранить override-файл в Git, чтобы все могли подхватить его.
- **В .gitignore:** если override — это что-то очень локальное (пароли, специфическая среда), можно не хранить его в репозитории.

### 3. Итоги

- **Секция `build:`** в Docker Compose даёт возможность собирать образы автоматически — удобнее, чем отдельно делать `docker build`.
    - Поддерживаются `context`, `dockerfile`, `args`, `target`, `cache_from` и т. д.
    - В результате Compose становится центром управления: сборка, запуск, сетевые настройки — всё в одном YAML.
- **docker-compose.override.yml** (и вообще несколько YAML-файлов) позволяют хранить разные конфигурации (prod/dev) без постоянного редактирования одного файла.
    - При `docker-compose up` файлы автоматически объединяются (если override-файл назван точно `docker-compose.override.yml`).
    - Явный флаг `-f` помогает комбинировать несколько файлов или переопределить имя override-файла.
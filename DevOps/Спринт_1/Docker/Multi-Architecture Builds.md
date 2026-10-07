## Цель

Научиться собирать Docker-образы для нескольких архитектур процессоров одновременно.

## Проблема

По умолчанию Docker собирает образ **только для вашей архитектуры**:

- На Intel/AMD (x86_64) → образ работает только на amd64
- На Mac M1/M2 (ARM) → образ работает только на arm64

**Проблемы:**

- Образ с MacBook M1 не запустится на обычном Linux-сервере
- Образ с x86 не запустится на Raspberry Pi или AWS Graviton
- Каждую архитектуру нужно собирать отдельно

**Решение:** Multi-architecture builds с Docker Buildx

## Что такое Multi-Arch

Multi-arch (multi-architecture) образы содержат версии для разных архитектур в одном образе.

**Пример:**

docker pull python:3.11-slim

Этот образ поддерживает:

- `linux/amd64` (Intel/AMD 64-bit)
- `linux/arm64` (ARM 64-bit)
- `linux/arm/v7` (ARM 32-bit)
- и другие...

Docker автоматически выбирает правильную версию для вашей системы!

## Основные архитектуры

|Архитектура|Альтернативные имена|Где используется|
|---|---|---|
|`linux/amd64`|x86_64, x64|Intel/AMD серверы, обычные ПК|
|`linux/arm64`|aarch64, arm64v8|Mac M1/M2, AWS Graviton, смартфоны|
|`linux/arm/v7`|armhf, armv7l|Raspberry Pi 2/3/4 (32-bit)|
|`linux/arm/v6`|armel, armv6l|Raspberry Pi Zero/1|
|`linux/386`|i386, x86|Старые 32-bit Intel|

**Самые важные для продакшена:**

- `linux/amd64` - основная архитектура серверов
- `linux/arm64` - растущая популярность (Apple Silicon, AWS Graviton дешевле)

## Docker Buildx

**Buildx** - расширение Docker для продвинутых возможностей сборки.

### Проверка установки:

docker buildx version

Если установлен, увидите версию. Buildx включен по умолчанию в Docker Desktop и современных версиях Docker Engine.

### Проверка поддержки multi-arch:

docker buildx ls

Вывод:

codecopy

```
NAME/NODE       DRIVER/ENDPOINT             STATUS  PLATFORMS
default *       docker
  default       default                     running linux/amd64, linux/arm64, linux/arm/v7, linux/arm/v6
```

**PLATFORMS** - список поддерживаемых архитектур.

## Создание билдера для multi-arch

### Шаг 1: Создать новый билдер

docker buildx create --name multiarch-builder --driver docker-container --bootstrap

**Параметры:**

- `--name multiarch-builder` - имя билдера
- `--driver docker-container` - использовать контейнер (поддерживает эмуляцию)
- `--bootstrap` - запустить сразу

### Шаг 2: Использовать созданный билдер

docker buildx use multiarch-builder

### Шаг 3: Проверить билдер

docker buildx inspect --bootstrap

Вывод должен показать:

codecopy

```
Name:   multiarch-builder
Driver: docker-container
Platforms: linux/amd64, linux/arm64, linux/arm/v7, linux/arm/v6, ...
```

## Сборка multi-arch образа

### Базовая команда:

bashcopy

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t myapp:latest \
  --push \
  .
```

**Параметры:**

- `--platform` - список архитектур через запятую
- `-t myapp:latest` - тег образа
- `--push` - пушить в registry (обязательно для multi-arch!)
- `.` - путь к Dockerfile

**ВАЖНО:**

- Multi-arch образы **нельзя** сохранить локально командой `build`
- Их **нужно** пушить в registry (Docker Hub, GitHub, приватный)
- Или использовать `--load` для одной архитектуры

## Практика: Сборка для нескольких архитектур

### Вариант 1: Локальная сборка (одна архитектура)

Для тестирования можно собрать только для текущей платформы:

bashcopy

```bash
docker buildx build \
  --platform linux/amd64 \
  -t multiarch-demo:amd64 \
  --load \
  .
```

`--load` - загрузить образ в локальный Docker (работает только для одной платформы).

### Вариант 2: Multi-arch с пушем в Docker Hub

**Требования:**

1. Аккаунт на Docker Hub
2. Залогиниться: `docker login`

**Сборка и пуш:**

bashcopy

```bash
# Логин в Docker Hub
docker login

# Сборка и пуш multi-arch образа
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t YOUR_DOCKERHUB_USERNAME/multiarch-demo:latest \
  --push \
  .
```

**Проверка:**

docker buildx imagetools inspect YOUR_DOCKERHUB_USERNAME/multiarch-demo:latest

Вывод покажет все архитектуры:

codecopy

```
Name:      docker.io/YOUR_USERNAME/multiarch-demo:latest
MediaType: application/vnd.docker.distribution.manifest.list.v2+json
Digest:    sha256:...

Manifests:
  Name:      docker.io/YOUR_USERNAME/multiarch-demo:latest@sha256:...
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/amd64

  Name:      docker.io/YOUR_USERNAME/multiarch-demo:latest@sha256:...
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/arm64
```

### Вариант 3: Сборка без пуша (экспорт в файл)

Можно экспортировать в tar для переноса:

bashcopy

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t multiarch-demo:latest \
  -o type=oci,dest=./multiarch-demo.tar \
  .
```

## Как это работает

### QEMU эмуляция

Buildx использует QEMU для эмуляции других архитектур.

**Проверить QEMU:**

docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

Эта команда устанавливает эмуляторы для разных архитектур.

**Проверить доступные эмуляторы:**

ls /proc/sys/fs/binfmt_misc/

### Manifest Lists

Multi-arch образы используют **manifest list** - специальный формат, который содержит ссылки на образы для разных платформ.

Когда вы делаете `docker pull myapp:latest`, Docker:

1. Скачивает manifest list
2. Определяет вашу архитектуру
3. Скачивает только нужный образ

## Проверка архитектуры образа

### Проверить образ в registry:

docker buildx imagetools inspect python:3.11-slim

Вывод покажет все поддерживаемые платформы.

### Проверить локальный образ:

docker image inspect myapp:latest --format '{{.Architecture}}'

### Проверить внутри контейнера:

docker run --rm myapp:latest uname -m

Вывод:

- `x86_64` - amd64
- `aarch64` - arm64
- `armv7l` - arm/v7

## Команды для работы с Buildx

bashcopy

```bash
# Список билдеров
docker buildx ls

# Создать билдер
docker buildx create --name mybuilder --driver docker-container --bootstrap

# Использовать билдер
docker buildx use mybuilder

# Удалить билдер
docker buildx rm mybuilder

# Информация о билдере
docker buildx inspect mybuilder

# Остановить билдер
docker buildx stop mybuilder

# Очистка кэша
docker buildx prune

# Вернуться к default билдеру
docker buildx use default
```

## Оптимизация multi-arch сборки

### 1. Использовать кэш

bashcopy

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=myapp:buildcache \
  --cache-to type=registry,ref=myapp:buildcache,mode=max \
  -t myapp:latest \
  --push \
  .
```

### 2. Собирать только нужные архитектуры

Не нужно собирать все архитектуры, если не используете:

bashcopy

```bash
# Только самые популярные
--platform linux/amd64,linux/arm64
```

### 3. Использовать base images с multi-arch

Выбирайте базовые образы, которые уже поддерживают multi-arch:

- ✅ `python:3.11-slim`
- ✅ `node:18-alpine`
- ✅ `nginx:alpine`
- ✅ `ubuntu:22.04`

## Практическое задание

### Задание 1: Собрать локально для своей архитектуры

bashcopy

```bash
cd d5

docker buildx build \
  --platform linux/amd64 \
  -t multiarch-demo:local \
  --load \
  .

docker run -d -p 8080:8080 multiarch-demo:local
```
## ![](https://ucarecdn.stepik.net/087c73df-e636-4da2-95e0-2b5a00ff5e38/)

В традиционном Dockerfile мы прописываем все шаги — от компиляции или установки зависимостей до финального запуска приложения. Проблема в том, что все инструменты сборки (компиляторы, dev-зависимости, кэш) оказываются в готовом образе, раздувая его. **Multi-stage builds** позволяют разделить `этап сборки` и `этап исполнения` (runtime) на разные _стадии_ `Dockerfile`. В итоге финальный образ содержит _только_ то, что нужно для запуска.

### Как это работает?

1. В Dockerfile мы пишем несколько `FROM` (несколько стадий).
    - Первая стадия (_builder stage_) содержит все инструменты (компиляторы, зависимости), чтобы собрать / скомпилировать ваше приложение.
    - Вторая (и последующие) стадии берут результат (бинарник, скомпилированный jar) из первой, но _не тащат_ весь компилятор и прочие зависимости.
2. В финальном легковесном образе остаётся лишь _необходимое для запуска_ (runtime). Никаких компиляторов, исходников, кэша.

### Пример: Go

```bash
# СТАДИЯ 1: сборка
FROM golang:1.18-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# СТАДИЯ 2: минимальный runtime
FROM alpine:3.17
COPY --from=builder /app/myapp /usr/local/bin/myapp
CMD ["myapp"]
```

- В `FROM golang:1.18-alpine` есть всё необходимое для компиляции Go-приложения: go build.
- После `go build -o myapp` у нас получается бинарник `myapp`.
- Во втором `FROM alpine:3.17` у нас чистая среда (без go-toolchain).
- `COPY --from=builder /app/myapp` копирует бинарник из builder стадии.
- Финальный образ очень маленький, ведь в нём нет компиляторов.

### Более продвинутый пример Go: использование scratch и статическая компиляция

Для Go-приложений мы можем создать ещё более минимальный образ, используя базовый образ `scratch` (пустой образ) и статическую компиляцию:

```bash
# СТАДИЯ 1: сборка с полной статической компиляцией
FROM golang:1.18-alpine AS builder
WORKDIR /app
COPY . .
# Статическая компиляция без CGO для создания полностью независимого бинарника
RUN CGO_ENABLED=0 GOOS=linux go build -a -ldflags '-extldflags "-static"' -o myapp

# СТАДИЯ 2: пустой scratch-образ
FROM scratch
# Копируем SSL-сертификаты, если приложение будет делать HTTPS-запросы
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/myapp /myapp
# Запускаем бинарник напрямую
CMD ["/myapp"]
```

Преимущества такого подхода:

- Экстремально маленький финальный образ (несколько MB вместо ~100MB)
- Нет базовой ОС, только ваш бинарник и сертификаты
- Повышенная безопасность — отсутствие потенциально уязвимых компонентов

### Пример: Node (production build)

В Node-проектах, при `production build` (React, Vue, Angular) можно собрать статику в одной стадии, а раздавать её через Nginx в другой.

```sql
# СТАДИЯ 1: сборка (Node)
FROM node:16-alpine AS build
WORKDIR /app
COPY package*.json .
RUN npm install
COPY . .
RUN npm run build

# СТАДИЯ 2: раздача (Nginx)
FROM nginx:1.21-alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

- Первая стадия (_build_): node + npm install + npm run build => создаёт финальный dist.
- Вторая: minimal Nginx-образ. Копируем результат `/app/dist` прямо в `/usr/share/nginx/html`.
- Финальный образ в итоге не содержит Node или npm — только файлы статики и Nginx.

### Оптимизированный Node-пример с кэшированием зависимостей

Для улучшения производительности сборки и более эффективного использования кэша Docker можно разделить установку зависимостей и сборку кода:

```bash
# СТАДИЯ 1: только зависимости
FROM node:16-alpine AS deps
WORKDIR /app
# Копируем только файлы зависимостей
COPY package.json package-lock.json ./
# Установка зависимостей
RUN npm ci --only=production

# СТАДИЯ 2: сборка на основе зависимостей
FROM node:16-alpine AS builder
WORKDIR /app
# Копируем установленные зависимости из первой стадии
COPY --from=deps /app/node_modules ./node_modules
# Теперь копируем исходный код
COPY . .
# Сборка
RUN npm run build

# СТАДИЯ 3: production образ
FROM nginx:1.21-alpine AS production
# Настройка Nginx (опционально)
COPY nginx.conf /etc/nginx/conf.d/default.conf
# Копирование только собранных файлов
COPY --from=builder /app/dist /usr/share/nginx/html
# Запуск Nginx
CMD ["nginx", "-g", "daemon off;"]
```

Преимущества этого подхода:

- При изменении только исходного кода (без изменения package.json) Docker переиспользует кэш первой стадии
- Лучшая изоляция — каждая стадия имеет свою конкретную ответственность
- Возможность выбора конкретной стадии при сборке с помощью `--target`

### Пример: Java (Spring Boot)

Для Java-приложений multi-stage builds также дают существенную экономию размера, разделяя JDK (для сборки) и JRE (для запуска):

```bash
# СТАДИЯ 1: сборка с Maven и полным JDK
FROM maven:3.8-openjdk-17 AS builder
WORKDIR /app
# Копируем только pom.xml для кэширования зависимостей
COPY pom.xml .
RUN mvn dependency:go-offline

# Копируем исходный код и собираем приложение
COPY src ./src
RUN mvn package -DskipTests

# СТАДИЯ 2: только JRE для запуска
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# Копируем только JAR-файл из builder стадии
COPY --from=builder /app/target/*.jar app.jar
# Запуск приложения
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Эффект: размер финального образа уменьшается с ~500MB до ~120MB.

### Как Docker знает, что копировать?

Инструкция `COPY --from=<stage> /path/in/stage /path/in/final` позволяет вытащить файлы из предыдущей стадии. Имя стадии задаётся после `AS`, например `FROM node:16-alpine AS build`. При копировании пишут `--from=build`.

Также можно использовать индекс стадии вместо имени (не рекомендуется, так как это менее наглядно):

```bash
# СТАДИЯ 0 (индексация с нуля)
FROM golang:1.18-alpine
# ... команды ...

# СТАДИЯ 1
FROM alpine:3.17
# Копирование из стадии 0 по индексу
COPY --from=0 /app/myapp /usr/local/bin/myapp
```

### Работа с аргументами сборки (ARG) в Multi-stage

В Multi-stage Dockerfile можно передавать аргументы между стадиями, но есть некоторые нюансы:

```bash
# Глобальный ARG (перед первым FROM)
ARG VERSION=latest

# СТАДИЯ 1
FROM golang:1.18-alpine AS builder
# Нужно снова объявить ARG внутри стадии
ARG VERSION
RUN echo "Building version $VERSION"
# ...

# СТАДИЯ 2
FROM alpine:3.17
# И здесь снова объявляем, если нужно использовать
ARG VERSION
RUN echo "Version: $VERSION" > /version.txt
# ...
```

Важно: каждый `FROM` создаёт новый контекст, поэтому аргумент нужно объявлять заново в каждой стадии

### Выбор конкретной стадии при сборке

Если у вас есть несколько стадий, вы можете указать, какую именно нужно собрать, с помощью флага `--target`:

```applescript
# Собрать только стадию builder
docker build --target builder -t myapp:builder .

# Собрать полный образ до финальной стадии
docker build -t myapp:latest .
```

Это полезно при создании образов для разработки, тестирования или отладки.

### Работа с секретами в Multi-stage builds

Часто при сборке нужны секреты (токены доступа к приватным репозиториям и т.д.), но их нельзя хранить в образе. Multi-stage builds помогают решить эту проблему:

```bash
# СТАДИЯ 1: используем секреты
FROM node:16-alpine AS builder
WORKDIR /app
COPY . .
# Используем секрет для доступа к приватному репозиторию
RUN --mount=type=secret,id=npm_token \
    cat /run/secrets/npm_token > .npmrc && \
    npm install && \
    rm .npmrc  # Удаляем файл с секретом

# СТАДИЯ 2: в этой стадии секрет не сохраняется
FROM node:16-alpine AS production
WORKDIR /app
# Копируем только нужные файлы, без .npmrc с секретом
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json .
CMD ["npm", "start"]
```

Сборка с использованием этого секрета:

```bash
docker build --secret id=npm_token,src=.npmrc -t myapp:latest .
```

### Общие рекомендации

- Старайтесь **именовать** стадии: `FROM golang:1.18 AS builder`, `FROM alpine:3.17 AS final`, это читабельнее.
- В builder-стадии можно использовать `RUN`, `COPY` столько угодно. В финальной же минимальный базовый образ.
- Если нужно более 2-х стадий (например, одна для тестов, вторая для сборки, третья для runtime) — всё возможно.

### Сравнение размеров и примеры экономии

|Язык/фреймворк|Без multi-stage|С multi-stage|Экономия|
|---|---|---|---|
|Go|~800MB (golang:1.18)|~10MB (scratch)|~99%|
|Node.js + React|~1GB (node:16)|~100MB (nginx:alpine)|~90%|
|Java (Spring)|~500MB (maven+JDK)|~120MB (JRE)|~75%|
|Python|~900MB (python)|~250MB (python:slim)|~70%|

### Отладка Multi-stage builds

Если что-то идёт не так при multi-stage сборке, используйте эти техники:

1. Соберите образ до конкретной стадии: `docker build --target builder -t debug-image .`
2. Запустите интерактивную оболочку для исследования: `docker run -it debug-image sh`
3. Проверьте размер каждого слоя: `docker history myapp:latest`
4. Используйте `docker image ls` для просмотра всех образов, включая промежуточные

### Итог

**Multi-stage builds** — мощный способ сократить размер образа и отделить этап разработки/сборки от этапа запуска. Особенно полезно для языков с компиляцией (Go, C++, Rust) или больших JS-фреймворков (React/Vue) где сборка статических файлов происходит в одном этапе, а раздача — в другом. Рекомендуется использовать для «production» образов, где важно, чтобы в финале остался только «runtime».
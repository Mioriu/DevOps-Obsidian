
В предыдущих уроках мы научились создавать и настраивать собственный Docker Registry. Теперь рассмотрим, как эффективно взаимодействовать с ним через API, управлять образами и интегрировать Registry в процессы непрерывной интеграции и доставки (CI/CD).

### Docker Registry API v2

Docker Registry предоставляет RESTful API, который позволяет управлять репозиториями и образами программно. Это открывает широкие возможности для автоматизации и интеграции с различными инструментами.

#### Базовые эндпоинты API

Рассмотрим основные эндпоинты, с которыми вы можете взаимодействовать:

- **Проверка API** — получение информации о доступности и версии API:
    
    ```bash
    curl -X GET https://registry.example.com:5000/v2/
    ```
    
- **Список репозиториев** — получение каталога всех доступных репозиториев:
    
    ```bash
    curl -X GET https://registry.example.com:5000/v2/_catalog
    ```
    
    Ответ будет содержать список репозиториев в формате JSON:
    
    ```json
    {"repositories":["app1", "app2", "nginx"]}
    ```
    
- **Список тегов** — получение всех тегов конкретного репозитория:
    
    ```bash
    curl -X GET https://registry.example.com:5000/v2/nginx/tags/list
    ```
    
    Ответ будет в формате:
    
    ```json
    {"name":"nginx","tags":["1.19", "latest", "stable"]}
    ```
    

#### Аутентификация в API

Если ваш Registry защищен аутентификацией (что рекомендуется), то к запросам нужно добавлять учетные данные:

```bash
curl -X GET https://registry.example.com:5000/v2/_catalog \
  --user username:password
```

Или через токен (Bearer Authentication):

```bash
# Сначала получаем токен
TOKEN=$(curl -s -H "Authorization: Basic $(echo -n 'username:password' | base64)" \
  "https://registry.example.com:5000/v2/token?service=registry.docker.io&scope=repository:nginx:pull" \
  | jq -r '.token')

# Используем токен для запроса
curl -H "Authorization: Bearer $TOKEN" \
  https://registry.example.com:5000/v2/nginx/tags/list
```

### Управление образами через API

API позволяет не только получать информацию, но и выполнять операции с образами.

#### Получение метаданных образа

Для работы с образом часто нужно получить его манифест и digest (хэш):

```bash
# Получение манифеста по тегу
curl -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  -X GET https://registry.example.com:5000/v2/nginx/manifests/latest \
  --user username:password
```

Обратите внимание на заголовок `Accept` — он указывает формат манифеста. В ответе будет заголовок `Docker-Content-Digest`, содержащий хэш образа.

#### Удаление образа

Для удаления образа нужно использовать его digest:

```bash
# Сначала получаем digest образа
DIGEST=$(curl -v --silent -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  -X GET https://registry.example.com:5000/v2/nginx/manifests/latest \
  --user username:password 2>&1 | grep Docker-Content-Digest | awk '{print $3}' | tr -d '\r')

# Затем удаляем образ по digest
curl -X DELETE https://registry.example.com:5000/v2/nginx/manifests/$DIGEST \
  --user username:password
```

Важно помнить, что для возможности удаления нужно включить соответствующую опцию при запуске Registry (`REGISTRY_STORAGE_DELETE_ENABLED=true`).

#### Очистка неиспользуемого пространства

После удаления манифестов необходимо запустить сборщик мусора для освобождения дискового пространства:

```bash
docker exec -it registry registry garbage-collect /etc/docker/registry/config.yml
```

Для автоматизации процесса очистки можно создать скрипт:

```bash
#!/bin/bash
# cleanup-registry.sh

# Переменные
REGISTRY_URL="https://registry.example.com:5000"
USERNAME="admin"
PASSWORD="password"
REPO="nginx"
TAG="old-tag"

# Получение digest
DIGEST=$(curl -s -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  --user $USERNAME:$PASSWORD \
  $REGISTRY_URL/v2/$REPO/manifests/$TAG | jq -r '.config.digest')

# Удаление образа
curl -X DELETE --user $USERNAME:$PASSWORD \
  $REGISTRY_URL/v2/$REPO/manifests/$DIGEST

# Запуск сборщика мусора
docker exec -it registry registry garbage-collect /etc/docker/registry/config.yml

echo "Образ $REPO:$TAG успешно удален и пространство очищено"
```

### Интеграция Docker Registry с CI/CD

Собственный Registry становится особенно полезным при интеграции с системами непрерывной интеграции и доставки.

#### Пример интеграции с GitLab CI/CD

Рассмотрим пример настройки пайплайна в `.gitlab-ci.yml` для сборки, публикации и деплоя образа:

```bash
stages:
  - build
  - test
  - deploy

variables:
  REGISTRY_URL: registry.example.com:5000
  IMAGE_NAME: $REGISTRY_URL/myapp
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

# Сборка образа
build:
  stage: build
  image: docker:20.10
  services:
    - docker:20.10-dind
  before_script:
    - echo $REGISTRY_PASSWORD | docker login $REGISTRY_URL -u $REGISTRY_USERNAME --password-stdin
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker push $IMAGE_NAME:$IMAGE_TAG
  after_script:
    - docker logout $REGISTRY_URL

# Тестирование образа
test:
  stage: test
  image: docker:20.10
  services:
    - docker:20.10-dind
  before_script:
    - echo $REGISTRY_PASSWORD | docker login $REGISTRY_URL -u $REGISTRY_USERNAME --password-stdin
  script:
    - docker pull $IMAGE_NAME:$IMAGE_TAG
    - docker run --rm $IMAGE_NAME:$IMAGE_TAG ./run_tests.sh
  after_script:
    - docker logout $REGISTRY_URL

# Деплой на продакшн
deploy:
  stage: deploy
  image: docker:20.10
  services:
    - docker:20.10-dind
  before_script:
    - echo $REGISTRY_PASSWORD | docker login $REGISTRY_URL -u $REGISTRY_USERNAME --password-stdin
  script:
    - docker pull $IMAGE_NAME:$IMAGE_TAG
    - docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
    - docker push $IMAGE_NAME:latest
    - curl -X POST https://deployment-hook.example.com/deploy
  after_script:
    - docker logout $REGISTRY_URL
  only:
    - main
```

В этом примере:

- CI система авторизуется в вашем Registry
- Собирает Docker-образ из исходного кода
- Публикует его в ваш Registry с тегом, основанным на хеше коммита
- Запускает тесты на собранном образе
- При успешном тестировании помечает образ как `latest` и уведомляет систему деплоя

#### Пример для GitHub Actions

Аналогичный пайплайн для GitHub Actions (`.github/workflows/docker-build.yml`):

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      
      - name: Login to private registry
        uses: docker/login-action@v1
        with:
          registry: registry.example.com:5000
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      
      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: |
            registry.example.com:5000/myapp:${{ github.sha }}
            registry.example.com:5000/myapp:latest
      
      - name: Deploy
        if: github.ref == 'refs/heads/main'
        run: |
          curl -X POST https://deployment-hook.example.com/deploy
```

#### Автоматизация очистки старых образов

Важной частью управления Registry является регулярная очистка устаревших образов. Вот пример скрипта для удаления старых тегов, который можно добавить в CI/CD пайплайн:

```bash
#!/bin/bash
# cleanup-old-images.sh

REGISTRY_URL="https://registry.example.com:5000"
USERNAME="admin"
PASSWORD="password"
REPO="myapp"
KEEP_LAST=5

# Получение всех тегов, сортировка и выбор старых
TAGS=$(curl -s --user $USERNAME:$PASSWORD \
  $REGISTRY_URL/v2/$REPO/tags/list | jq -r '.tags[]' | sort -r | tail -n +$((KEEP_LAST+1)))

for TAG in $TAGS; do
  echo "Удаление тега $REPO:$TAG"
  
  # Получение digest для тега
  DIGEST=$(curl -s -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
    --user $USERNAME:$PASSWORD \
    $REGISTRY_URL/v2/$REPO/manifests/$TAG | jq -r '.config.digest')
  
  # Удаление образа
  curl -X DELETE --user $USERNAME:$PASSWORD \
    $REGISTRY_URL/v2/$REPO/manifests/$DIGEST
done

# Запуск сборщика мусора
docker exec -it registry registry garbage-collect /etc/docker/registry/config.yml

echo "Очистка завершена, оставлены последние $KEEP_LAST тегов"
```

Этот скрипт можно запускать как отдельный job в CI/CD пайплайне или по расписанию через cron.

#### Webhook для автоматического обновления

Docker Registry поддерживает механизм уведомлений через webhook. Это позволяет автоматически реагировать на события в Registry (например, загрузку нового образа).

Пример настройки webhook в конфигурации Registry:

```yaml
version: '3'

services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    environment:
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_NOTIFICATIONS_ENDPOINTS_0_NAME: webhook
      REGISTRY_NOTIFICATIONS_ENDPOINTS_0_URL: https://deployment.example.com/hooks/registry
      REGISTRY_NOTIFICATIONS_ENDPOINTS_0_HEADERS_Authorization: "Bearer secrettoken"
      REGISTRY_NOTIFICATIONS_ENDPOINTS_0_TIMEOUT: 500ms
      REGISTRY_NOTIFICATIONS_ENDPOINTS_0_THRESHOLD: 5
      REGISTRY_NOTIFICATIONS_ENDPOINTS_0_BACKOFF: 1s
    volumes:
      - ./data:/var/lib/registry
      - ./certs:/certs
      - ./auth:/auth
    restart: always
```

На стороне сервера деплоя можно реализовать обработчик webhook, который будет обновлять приложение при получении уведомления о новом образе:

```javascript
// Простой пример на Express.js
const express = require('express');
const { exec } = require('child_process');
const app = express();

app.use(express.json());

app.post('/hooks/registry', (req, res) => {
  const authHeader = req.headers.authorization;
  
  // Проверка авторизации
  if (authHeader !== 'Bearer secrettoken') {
    return res.status(401).send('Unauthorized');
  }
  
  // Получение данных о событии
  const events = req.body.events;
  
  for (const event of events) {
    if (event.action === 'push' && event.target.repository === 'myapp') {
      console.log(`Получено уведомление о новом образе: ${event.target.repository}:${event.target.tag}`);
      
      // Запуск скрипта деплоя
      exec('./deploy.sh', (error, stdout, stderr) => {
        if (error) {
          console.error(`Ошибка деплоя: ${error}`);
          return;
        }
        console.log(`Деплой успешно выполнен: ${stdout}`);
      });
    }
  }
  
  res.status(200).send('Webhook обработан');
});

app.listen(3000, () => {
  console.log('Webhook-сервер запущен на порту 3000');
});
```

### Продвинутые сценарии использования

#### Мульти-стейдж сборка и оптимизация образов

Для эффективной работы CI/CD с Registry важно оптимизировать размер образов. Пример мульти-стейдж сборки в Dockerfile:

```sql
# Стадия сборки
FROM node:14-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Финальный образ
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### Версионирование и стратегии тегирования

Разумная стратегия тегирования образов в Registry упрощает управление и откат изменений:

```bash
# В CI/CD пайплайне
VERSION=$(cat VERSION)
GIT_COMMIT=$(git rev-parse --short HEAD)
BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Теги для образа
docker build -t $REGISTRY_URL/myapp:latest .
docker tag $REGISTRY_URL/myapp:latest $REGISTRY_URL/myapp:$VERSION
docker tag $REGISTRY_URL/myapp:latest $REGISTRY_URL/myapp:$VERSION-$GIT_COMMIT
docker tag $REGISTRY_URL/myapp:latest $REGISTRY_URL/myapp:$GIT_COMMIT

# Публикация всех тегов
docker push $REGISTRY_URL/myapp:latest
docker push $REGISTRY_URL/myapp:$VERSION
docker push $REGISTRY_URL/myapp:$VERSION-$GIT_COMMIT
docker push $REGISTRY_URL/myapp:$GIT_COMMIT
```

#### Зеркалирование и кэширование публичных образов

Собственный Registry можно использовать для кэширования образов из Docker Hub или других публичных реестров:

```yaml
version: '3'

services:
  registry-mirror:
    image: registry:2
    ports:
      - "5000:5000"
    environment:
      REGISTRY_PROXY_REMOTEURL: https://registry-1.docker.io
    volumes:
      - ./mirror-data:/var/lib/registry
    restart: always
```

После настройки зеркала нужно добавить его в конфигурацию Docker на клиентских машинах (`/etc/docker/daemon.json`):

```json
{
  "registry-mirrors": ["https://registry-mirror.example.com:5000"]
}
```
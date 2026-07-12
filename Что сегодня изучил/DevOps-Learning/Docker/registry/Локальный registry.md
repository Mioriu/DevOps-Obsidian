## Создание собственного Registry

В предыдущем уроке мы рассмотрели **Docker Hub** и другие публичные реестры. Но что делать, если вам нужен полный контроль над хранением образов, или вы хотите работать в закрытой сети? В этом случае вам поможет **собственный Docker Registry** — локальный сервер для хранения и распространения ваших Docker-образов.

### Зачем нужен собственный Registry?

Перед тем как углубиться в техническую реализацию, давайте рассмотрим, в каких случаях имеет смысл разворачивать собственный реестр:

- **Приватность** — хранение конфиденциальных образов в контролируемой среде
- **Скорость** — снижение времени доставки образов в локальной сети
- **Экономия трафика** — загрузка образов один раз и локальное распространение
- **Отказоустойчивость** — работа без зависимости от внешних сервисов
- **CI/CD интеграция** — встраивание Registry в конвейеры непрерывной интеграции

### Как устроен Docker Registry

Docker Registry — это отдельный проект с открытым исходным кодом, который предоставляет API для хранения и распространения образов Docker. Он сам поставляется как Docker-образ, что упрощает его развёртывание и управление.

### Базовый запуск Registry

Развернуть простой вариант Docker Registry очень легко. Достаточно запустить официальный образ `registry:2`:

```applescript
docker run -d -p 5000:5000 --name registry registry:2
```

После запуска вы получите работающий реестр, доступный по адресу `localhost:5000`. Теперь вы можете:

1. **Пометить (tag)** образ для отправки в ваш реестр:
    
    ```bash
    docker tag nginx:latest localhost:5000/my-nginx:latest
    ```
    
2. **Отправить (push)** образ в ваш реестр:
    
    ```perl
    docker push localhost:5000/my-nginx:latest
    ```
    
3. **Загрузить (pull)** образ из вашего реестра:
    
    ```bash
    docker pull localhost:5000/my-nginx:latest
    ```
    

Проверить список образов в реестре можно с помощью API-запроса:

```bash
curl -X GET http://localhost:5000/v2/_catalog
```

Этот запрос вернёт список репозиториев в формате JSON:

```json
{"repositories":["my-nginx"]}
```

### Постоянное хранение данных

По умолчанию Registry хранит все образы в своей внутренней файловой системе контейнера. Это значит, что при удалении контейнера все образы будут потеряны. Для постоянного хранения данных нужно использовать **volume**:

```haskell
docker run -d -p 5000:5000 --name registry \
  -v registry-data:/var/lib/registry \
  registry:2
```

Этот вариант создаёт именованный том `registry-data`, который сохраняется даже после удаления контейнера.

### Настройка безопасности и TLS

Наш базовый реестр работает без шифрования (HTTP), что подходит только для тестирования на локальной машине. Docker по умолчанию блокирует push/pull операции с небезопасными реестрами, кроме `localhost`.

Для использования реестра в реальной среде **обязательно нужно настроить TLS шифрование**:

1. **Подготовка сертификатов**
    
    Вам понадобятся SSL-сертификат и приватный ключ. Для тестовой среды можно создать самоподписанный сертификат:
    
    ```csharp
    # Создание директории для сертификатов
    mkdir -p certs
    cd certs
    
    # Генерация приватного ключа
    openssl genrsa -out domain.key 2048
    
    # Создание самоподписанного сертификата 
    # (замените registry.example.com на ваше доменное имя)
    openssl req -new -x509 -sha256 -key domain.key -out domain.crt -days 365 \
      -subj "/CN=registry.example.com"
    ```
    
2. **Запуск Registry с поддержкой HTTPS**
    
    ```bash
    docker run -d -p 5000:5000 --name registry \
      -v registry-data:/var/lib/registry \
      -v $(pwd)/certs:/certs \
      -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
      -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
      registry:2
    ```
    
    Теперь ваш реестр работает по HTTPS, но для самоподписанных сертификатов клиентам нужно дополнительно:
    
    - Добавить сертификат в доверенные на клиентских машинах или
    - Настроить Docker для работы с небезопасным реестром:
        
        ```bash
        # Linux - добавление в /etc/docker/daemon.json
        {
          "insecure-registries": ["registry.example.com:5000"]
        }
        
        # Рестарт демона Docker
        sudo systemctl restart docker
        ```
        

### Ограничение доступа

Для защиты вашего реестра необходимо настроить аутентификацию. Самый простой способ — это **базовая HTTP-аутентификация**:

1. **Создание файла с учётными данными**
    
    ```bash
    # Установка htpasswd (входит в пакет apache2-utils)
    sudo apt-get install apache2-utils # для Debian/Ubuntu
    # или
    sudo yum install httpd-tools # для CentOS/RHEL
    
    # Создание директории для аутентификации
    mkdir -p auth
    
    # Создание файла с пользователем (замените username и password)
    htpasswd -Bc auth/htpasswd username
    # Введите пароль дважды при запросе
    ```
    
2. **Запуск Registry с аутентификацией**
    
    ```bash
    docker run -d -p 5000:5000 --name registry \
      -v registry-data:/var/lib/registry \
      -v $(pwd)/certs:/certs \
      -v $(pwd)/auth:/auth \
      -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
      -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
      -e REGISTRY_AUTH=htpasswd \
      -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
      -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
      registry:2
    ```
    
3. **Аутентификация для push/pull**
    
    Теперь перед работой с реестром нужно аутентифицироваться:
    
    ```nginx
    docker login registry.example.com:5000
    # Введите имя пользователя и пароль
    ```
    
    После успешной аутентификации Docker сохранит учётные данные локально, и вы сможете выполнять push/pull операции.
    

### Настройка через docker-compose

Для более удобного управления реестром лучше использовать Docker Compose. Вот пример `docker-compose.yml` с полной конфигурацией безопасного реестра:

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
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_STORAGE_DELETE_ENABLED: "true"  # Включает возможность удаления образов
    volumes:
      - ./data:/var/lib/registry
      - ./certs:/certs
      - ./auth:/auth
    restart: always
```

### Управление образами в Registry

Docker Registry API v2 позволяет не только хранить образы, но и управлять ими:

1. **Просмотр списка репозиториев**
    
    ```bash
    curl -X GET https://registry.example.com:5000/v2/_catalog \
      --user username:password
    ```
    
2. **Просмотр тегов конкретного репозитория**
    
    ```bash
    curl -X GET https://registry.example.com:5000/v2/my-nginx/tags/list \
      --user username:password
    ```
    
3. **Удаление образа** (требует включения REGISTRY_STORAGE_DELETE_ENABLED)
    
    ```bash
    # Сначала получить digest (хеш) образа
    DIGEST=$(curl -v --silent -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
      -X GET https://registry.example.com:5000/v2/my-nginx/manifests/latest \
      --user username:password 2>&1 | grep Docker-Content-Digest | awk '{print $3}' | tr -d '\r')
    
    # Затем удалить манифест по хешу
    curl -X DELETE https://registry.example.com:5000/v2/my-nginx/manifests/$DIGEST \
      --user username:password
    ```
    

Важно отметить, что удаление манифеста не очищает фактически используемое дисковое пространство. Для этого нужно запустить сборщик мусора:

```bash
docker exec -it registry registry garbage-collect /etc/docker/registry/config.yml
```

### Расширенные возможности

Docker Registry поддерживает множество дополнительных опций, которые могут быть полезны в продакшн-окружении:

1. **Хранилище данных**: вместо локальной файловой системы можно использовать облачные сервисы:
    
    ```applescript
    # Пример настройки для Amazon S3
    docker run -d -p 5000:5000 --name registry \
      -e REGISTRY_STORAGE=s3 \
      -e REGISTRY_STORAGE_S3_REGION=us-east-1 \
      -e REGISTRY_STORAGE_S3_BUCKET=my-registry-bucket \
      -e REGISTRY_STORAGE_S3_ACCESSKEY=AKIAIOSFODNN7EXAMPLE \
      -e REGISTRY_STORAGE_S3_SECRETKEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
      registry:2
    ```
    
2. **Уведомления**: можно настроить отправку уведомлений о событиях (push, pull, delete) на внешние системы:
    
    ```bash
    # Пример настройки webhook
    -e REGISTRY_NOTIFICATIONS_ENDPOINTS_0_NAME=webhook \
    -e REGISTRY_NOTIFICATIONS_ENDPOINTS_0_URL=https://webhook.example.com/notify \
    -e REGISTRY_NOTIFICATIONS_ENDPOINTS_0_HEADERS_Authorization="Bearer secret-token"
    ```
    
3. **Кэширование**: Registry может работать как кэширующий прокси для Docker Hub или других реестров:
    
    ```bash
    -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io
    ```
    

### Производительные альтернативы

Официальный Docker Registry хорош для небольших и средних команд, но для крупных организаций существуют более функциональные альтернативы:

- **Harbor** — корпоративный реестр с веб-интерфейсом, скетированием безопасности, управлением ролями и репликацией
- **Nexus Repository OSS** — хранилище не только для Docker, но и для множества других форматов (Maven, npm, PyPI и т.д.)
- **GitLab Container Registry** — встроенный в GitLab реестр с интеграцией с CI/CD

## Итог

- **Docker Registry** позволяет создать собственное хранилище образов с полным контролем
- Для **продакшн-использования** обязательно настройте TLS и аутентификацию
- **Постоянное хранение** обеспечивается с помощью volumes или внешних хранилищ
- Docker Registry предоставляет **API** для управления образами и интеграции с другими системами
- Для **корпоративного использования** стоит рассмотреть альтернативы с расширенными возможностями

Создав собственный Docker Registry, вы получаете полный контроль над распространением образов в своей инфраструктуре, что критично для предприятий с высокими требованиями к безопасности и производительности.
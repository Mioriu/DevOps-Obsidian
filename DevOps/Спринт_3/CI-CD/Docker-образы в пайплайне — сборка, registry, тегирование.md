## Зачем собирать образы в CI

Тут, в принципе, всё просто. Цепочка такая: разработчик пушит код → пайплайн собирает Docker-образ → пушит его в registry → на проде или на каком-то стенде этот образ выкачивается и запускается. Вот эта цепочка — по сути, основа CI/CD в контейнерном мире.

![Docker-образы в CI: git push → build → registry → stand](https://offers.prostodevops.ru/diagrams/ci-cd/docker-pipeline.png)Docker-образы в CI: git push → build → registry → stand

Собирать образы руками на ноутбуке и пушить — это, конечно, не вариант. Сборка должна быть воспроизводимой, то есть один и тот же коммит всегда даёт один и тот же образ. Плюс в CI можно прогнать тесты, линтер, сканер безопасности — и только если всё ок, собрать образ. То есть, грубо говоря, у вас появляется гарантия, что в registry попадает только проверенный образ.

## image и services в .gitlab-ci.yml

Прежде чем говорить про сборку, давайте разберёмся с двумя директивами, которые вы будете постоянно видеть в пайплайнах.

### image

`image` — это Docker-образ, внутри которого выполняется ваша джоба. То есть, раннер с Docker executor берёт этот образ, создаёт из него контейнер и запускает ваш `script` внутри:

yamlcopy

```yaml
test:  image: python:3.11-slim  script:    - pip install -r requirements.txt    - pytest
```

Если у вас Go — ставите `image: golang:1.21`, для Node.js — `image: node:20-alpine`, и так далее. Каждая джоба может использовать свой образ, это удобно.

### services

Это дополнительные контейнеры, которые поднимаются рядом с основным. Нужны, когда вашей джобе, допустим, требуется база данных для интеграционных тестов, или Redis, или что-то ещё:

yamlcopy

```yaml
integration-tests:  image: python:3.11-slim  services:    - name: postgres:15      alias: db      variables:        POSTGRES_DB: testdb        POSTGRES_USER: test        POSTGRES_PASSWORD: test    - name: redis:7-alpine      alias: cache  variables:    DATABASE_URL: "postgresql://test:test@db:5432/testdb"    REDIS_URL: "redis://cache:6379"  script:    - pip install -r requirements.txt    - pytest tests/integration/
```

Контейнеры из `services` доступны по `alias`. То есть, PostgreSQL доступен по хосту `db`, Redis — по `cache`. Всё это живёт в одной сети, видит друг друга. После завершения джобы все контейнеры уничтожаются.

![GitLab CI services: Postgres + Redis рядом с основным контейнером](https://offers.prostodevops.ru/diagrams/ci-cd/docker-services.png)GitLab CI services: Postgres + Redis рядом с основным контейнером

## Сборка Docker-образов — DinD vs Kaniko

Так, а вот тут начинается интересное. Docker executor. И вот внутри этого контейнера вам нужно запустить `docker build`. То есть, Docker внутри Docker. По умолчанию это просто так не работает — контейнер не имеет доступа к Docker-демону хоста.

Есть два основных подхода, как эту проблему решить: Docker-in-Docker и Kaniko.

### Docker-in-Docker (DinD)

Суть — рядом с вашим основным контейнером поднимается ещё один контейнер, service, внутри которого крутится Docker-демон. И ваш основной контейнер подключается к нему и через него делает `docker build`, `docker push`, всё это:

yamlcopy

```yaml
build-image:  image: docker:24  services:    - docker:24-dind  variables:    DOCKER_TLS_CERTDIR: "/certs"  script:    - docker build -t myapp:latest .    - docker push myapp:latest
```

![Docker-in-Docker (DinD): клиент docker:24 → docker:24-dind демон](https://offers.prostodevops.ru/diagrams/ci-cd/docker-dind.png)Docker-in-Docker (DinD): клиент docker:24 → docker:24-dind демон

Но для этого раннер должен быть настроен с `privileged = true` в `config.toml`:

tomlcopy

```toml
[[runners]]  [runners.docker]    privileged = true
```

`privileged` — это, по сути, даёт контейнеру почти полный доступ к хостовой системе. С точки зрения безопасности — риск, да. Поэтому DinD обычно используют на своих, доверенных раннерах, где вы контролируете, кто что запускает.

### Kaniko

Kaniko — это альтернативный способ. Он собирает Docker-образы без Docker-демона вообще. Не требует привилегированного режима, работает в обычном контейнере:

yamlcopy

```yaml
build-image:  image:    name: gcr.io/kaniko-project/executor:v1.19.2-debug    entrypoint: [""]  script:    - >      /kaniko/executor      --context "${CI_PROJECT_DIR}"      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"      --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
```

То есть, Kaniko читает Dockerfile, собирает образ слой за слоем и пушит в registry. Всё это без Docker-демона, без privileged. Синтаксис непривычный, конечно — это не `docker build`, а `/kaniko/executor` с кучей флагов. Но зато безопасно.

### Что выбрать

![DinD vs Kaniko: сравнение для сборки Docker-образов в CI](https://offers.prostodevops.ru/diagrams/ci-cd/docker-build-tools.png)DinD vs Kaniko: сравнение для сборки Docker-образов в CI

В принципе, если у вас свои раннеры и вы контролируете, кто на них что запускает — DinD проще и привычнее. Если shared раннеры или безопасность в приоритете — Kaniko.

## Registry — где хранить образы

Окей, собрали образ. Его нужно где-то хранить. Это место называется container registry. По сути, как мы уже говорили в теме про Docker, — мы собрали артефакт, и этот артефакт нужно куда-то положить.

### GitLab Container Registry

Встроен в GitLab. Если у вас self-hosted GitLab — нужно включить в настройках. На gitlab.com — работает из коробки, ничего делать не надо.

yamlcopy

```yaml
before_script:  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRYscript:  - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .  - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

`$CI_REGISTRY_USER`, `$CI_REGISTRY_PASSWORD`, `$CI_REGISTRY`, `$CI_REGISTRY_IMAGE` — это всё предопределённые переменные GitLab. Ничего руками прописывать не нужно, GitLab сам подставляет.

### Nexus Repository

В компаниях, особенно в российских, Nexus стоит практически везде. Self-hosted, все данные внутри контура, никаких внешних зависимостей. Плюс Nexus хранит не только Docker-образы — туда же можно складывать Maven-артефакты, npm-пакеты, PyPI, вот это всё. Всё в одном месте.

yamlcopy

```yaml
variables:  NEXUS_REGISTRY: nexus.company.internal:5000 before_script:  - docker login -u $NEXUS_USER -p $NEXUS_PASSWORD $NEXUS_REGISTRYscript:  - docker build -t $NEXUS_REGISTRY/myapp:$CI_COMMIT_SHORT_SHA .  - docker push $NEXUS_REGISTRY/myapp:$CI_COMMIT_SHORT_SHA
```

`$NEXUS_USER` и `$NEXUS_PASSWORD` задаёте через CI/CD Variables в GitLab — masked, protected.

### Harbor

Ещё один вариант — Harbor. Open-source registry, заточен под enterprise. Сканирование образов на уязвимости, подписи, репликация между дата-центрами, квоты. Встречается в компаниях с несколькими ДЦ, часто в связке с Kubernetes.

В принципе, логика для любого registry одинаковая: `docker login` → `docker build -t registry/image:tag` → `docker push`. Меняется только адрес и креды.

## Тегирование образов

Тег — это, по сути, версия вашего образа. И от того, как вы тегируете образы, зависит, сможете ли вы потом нормально откатиться, понять, что задеплоено, отдебажить проблему.

### Плохо: только latest

- docker build -t myapp:latest .

`latest` перезаписывается каждый раз. Вы не знаете, из какого коммита собран образ. Откатиться невозможно — предыдущий latest уже затёрт. На проде так делать нельзя.

### Правильно: commit SHA

- docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .

Каждый коммит — уникальный тег. `myapp:a1b2c3d4`. Всегда видно, из какого коммита собран. Нужно откатиться — просто деплоите предыдущий тег. Это, в принципе, минимально правильный подход.

### Правильно: semver по git-тегам

yamlcopy

```yaml
build-release:  script:    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG .  rules:    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
```

Поставили git-тег `v1.2.3` → образ получает Docker-тег `myapp:v1.2.3`. Major.minor.patch — сразу понятно, что изменилось.

### Комбинированный подход

На практике обычно ставят несколько тегов одновременно:

yamlcopy

```yaml
script:  # Всегда тегируем коммитом  - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .  - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA   # Если это main — дополнительно latest  - |    if [ "$CI_COMMIT_BRANCH" == "main" ]; then      docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA $CI_REGISTRY_IMAGE:latest      docker push $CI_REGISTRY_IMAGE:latest    fi   # Если это тег — дополнительно версией  - |    if [ -n "$CI_COMMIT_TAG" ]; then      docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG      docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG    fi
```

diagramcopy

```
  Коммит a1b2c3d4 в feature/login:
    → myapp:a1b2c3d4

  Коммит f5e6d7c8 в main:
    → myapp:f5e6d7c8
    → myapp:latest

  Git-тег v1.2.3:
    → myapp:9a8b7c6d
    → myapp:v1.2.3
    → myapp:latest
```

То есть, commit SHA — всегда. latest — для удобства, чтобы можно было быстро скачать последнюю версию. Но на проде деплоим строго по конкретному тегу — SHA или semver. Никогда по latest, потому что непонятно, что за версия, и откатиться невозможно.
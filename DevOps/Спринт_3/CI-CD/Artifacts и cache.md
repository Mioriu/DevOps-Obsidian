## Artifacts и cache — в чём разница

Оба механизма сохраняют файлы между джобами. Оба описываются в `.gitlab-ci.yml`. Но работают по-разному и нужны для разных задач.

Короткая версия:

- **Artifacts** — результат работы джобы. Передаются между стейджами, хранятся в GitLab, можно скачать.
- **Cache** — ускорение повторных запусков. Сохраняет файлы между пайплайнами, хранится на раннере, может потеряться — и это нормально.

![Artifacts vs Cache в GitLab CI: разница и применение](https://offers.prostodevops.ru/diagrams/ci-cd/artifacts-vs-cache.png)Artifacts vs Cache в GitLab CI: разница и применение

## Artifacts

### Базовый пример

yamlcopy

```yaml
build:  stage: build  script:    - npm install    - npm run build  artifacts:    paths:      - dist/
```

Джоба `build` собирает приложение, результат попадает в `dist/`. Блок `artifacts` говорит GitLab: «Сохрани директорию `dist/` — она понадобится дальше».

Теперь любая джоба в следующих стейджах автоматически получит эту директорию:

yamlcopy

```yaml
test:  stage: test  script:    - ls dist/          # dist/ уже здесь, скачался из артефактов build-джобы    - npm test deploy:  stage: deploy  script:    - scp -r dist/* user@server:/var/www/
```

### Как это работает

![Artifacts flow: build → test → deploy через GitLab](https://offers.prostodevops.ru/diagrams/ci-cd/artifacts-flow.png)Artifacts flow: build → test → deploy через GitLab

Артефакты загружаются в GitLab после завершения джобы и скачиваются на раннер перед запуском зависимой джобы. Именно поэтому `image` в разных джобах может быть разным — артефакты передаются через GitLab, а не через файловую систему раннера.

### Срок хранения

По умолчанию артефакты хранятся 30 дней (настраивается администратором). Можно задать свой срок:

yamlcopy

```yaml
build:  artifacts:    paths:      - dist/    expire_in: 1 week       # удалить через неделю
```

Варианты: `30 min`, `2 hrs`, `1 day`, `1 week`, `1 month`, `never`.

Для тестовых отчётов — `1 day` достаточно. Для релизных артефактов — `never` или длительный срок.

### Артефакты для тестовых отчётов

GitLab умеет парсить тестовые отчёты и показывать результаты прямо в merge request:

yamlcopy

```yaml
test:  script:    - pytest --junitxml=report.xml  artifacts:    reports:      junit: report.xml
```

В merge request появится вкладка с результатами тестов — какие прошли, какие упали, без необходимости лезть в логи.

Поддерживаемые форматы отчётов: `junit` (тесты), `coverage_report` (покрытие кода), `sast` (статический анализ безопасности), `dependency_scanning`, `container_scanning` и другие.

### Артефакты + needs

Если используете `needs` (DAG), артефакты скачиваются только от тех джоб, которые указаны в `needs`:

yamlcopy

```yaml
test-backend:  needs:    - job: build-backend      artifacts: true        # скачать артефакты build-backend  script:    - pytest
```

`artifacts: true` — дефолт, можно не писать. Но если артефакты от зависимой джобы не нужны:

yamlcopy

```yaml
deploy:  needs:    - job: test-backend      artifacts: false       # тесты ничего не создают, не качай    - job: build-backend      artifacts: true        # а вот сборку — качай
```

Это ускоряет пайплайн — не скачиваются лишние файлы.

## Cache

### Базовый пример

yamlcopy

```yaml
install-deps:  stage: build  cache:    paths:      - node_modules/  script:    - npm install    - npm run build
```

Первый запуск: `npm install` скачивает все зависимости, создаёт `node_modules/`. После завершения джобы `node_modules/` сохраняется в кэш.

Второй запуск: перед выполнением `script` раннер восстанавливает `node_modules/` из кэша. `npm install` видит, что зависимости уже есть, и доустанавливает только изменившиеся. Вместо 2 минут — 15 секунд.

### Cache key

По умолчанию кэш шарится между всеми ветками. Это не всегда хорошо — ветка `feature-x` может закэшировать свои зависимости, а ветка `main` получит их вместо своих.

Решение — `key`:

yamlcopy

```yaml
cache:  key: ${CI_COMMIT_REF_SLUG}       # отдельный кэш для каждой ветки  paths:    - node_modules/
```

Ещё лучше — привязать к lock-файлу:

yamlcopy

```yaml
cache:  key:    files:      - package-lock.json          # кэш инвалидируется при изменении lock-файла  paths:    - node_modules/
```

Если `package-lock.json` не изменился — берём кэш. Изменился (добавили новую зависимость) — кэш пересоздаётся. Это правильный подход.

Для Python:

yamlcopy

```yaml
cache:  key:    files:      - requirements.txt  paths:    - .pip-cache/  script:    - pip install --cache-dir .pip-cache -r requirements.txt
```

### Cache policy

yamlcopy

```yaml
cache:  paths:    - node_modules/  policy: pull                     # только читать кэш, не обновлять
```

Политики:

- **pull-push** (дефолт) — скачать кэш перед джобой, загрузить обновлённый после.
- **pull** — только скачать, не обновлять. Для джоб, которые не меняют зависимости (тесты, линтер).
- **push** — только загрузить, не скачивать. Для джобы, которая специально пересоздаёт кэш.

Типичный паттерн:

yamlcopy

```yaml
install:  stage: build  cache:    paths:      - node_modules/    policy: pull-push              # эта джоба обновляет кэш  script:    - npm ci test:  stage: test  cache:    paths:      - node_modules/    policy: pull                   # эта только читает, не перезаписывает  script:    - npm test
```

### Когда кэш пропадает

Кэш хранится на раннере (или в S3, если настроен distributed cache). Он может пропасть:

- Раннер пересоздан или почищен.
- Джоба попала на другой раннер (в кластере несколько раннеров).
- Ключ кэша изменился.

Пайплайн **не должен ломаться** при отсутствии кэша. Кэш — это оптимизация. Если его нет, `npm install` просто скачает всё заново. Медленнее, но работает.

## Variables — переменные

Переменные в GitLab CI позволяют параметризовать пайплайн. Не хардкодить URL-ы, пароли, флаги, а передавать их через переменные.

### Где задаются

**В .gitlab-ci.yml:**

yamlcopy

```yaml
variables:  APP_ENV: production  DOCKER_REGISTRY: registry.example.com build:  script:    - echo "Building for $APP_ENV"    - docker build -t $DOCKER_REGISTRY/myapp .
```

**В GitLab UI** (Settings → CI/CD → Variables):

Для секретов — паролей, токенов, ключей. Никогда не кладите секреты в `.gitlab-ci.yml` — файл в репозитории, его видят все.

**На уровне группы:**

Переменная, заданная на уровне группы, доступна во всех проектах этой группы. Удобно для общих настроек: адрес registry, общие креды для Nexus и т.д.

### Типы переменных в UI

**Variable** (дефолт) — обычная переменная, доступна как `$VARIABLE_NAME`.

**File** — GitLab создаёт временный файл с содержимым переменной. `$VARIABLE_NAME` будет содержать путь к файлу, а не значение. Используется для сертификатов, ключей, kubeconfig:

yamlcopy

```yaml
deploy:  script:    - kubectl --kubeconfig=$KUBECONFIG apply -f manifests/    # $KUBECONFIG = /tmp/gitlab_ci_xxxx (путь к файлу с kubeconfig)
```

### Mask и Protect

**Masked** — значение переменной не показывается в логах. Вместо пароля в логе будет `[MASKED]`. Обязательно для любых секретов.

**Protected** — переменная доступна только в джобах, запущенных из protected веток (main, release/*) или protected тегов. Креды от прода выдаются только при деплое из main — логично.

![Masked + Protected переменные в GitLab CI](https://offers.prostodevops.ru/diagrams/ci-cd/vars-masked-protected.png)Masked + Protected переменные в GitLab CI

### Предопределённые переменные

GitLab автоматически создаёт десятки переменных в каждой джобе. Самые полезные:

yamlcopy

```yaml
# Информация о коммите$CI_COMMIT_SHA              # полный хэш коммита: a1b2c3d4e5f6...$CI_COMMIT_SHORT_SHA        # короткий хэш: a1b2c3d4$CI_COMMIT_BRANCH           # имя ветки: main, feature/login$CI_COMMIT_TAG              # тег, если пайплайн запущен по тегу: v1.2.3$CI_COMMIT_MESSAGE          # сообщение коммита # Информация о пайплайне$CI_PIPELINE_ID             # уникальный ID пайплайна: 123456$CI_PIPELINE_SOURCE         # что запустило: push, merge_request_event, web, schedule$CI_JOB_ID                  # ID текущей джобы$CI_JOB_NAME                # имя текущей джобы # Информация о проекте$CI_PROJECT_NAME            # имя проекта: myapp$CI_PROJECT_NAMESPACE       # namespace: mycompany/backend$CI_PROJECT_DIR             # рабочая директория: /builds/mycompany/myapp$CI_REGISTRY                # адрес GitLab Registry: registry.gitlab.com$CI_REGISTRY_IMAGE          # полный путь образа: registry.gitlab.com/mycompany/myapp # Merge request$CI_MERGE_REQUEST_IID       # номер MR$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME   # ветка-источник$CI_MERGE_REQUEST_TARGET_BRANCH_NAME   # целевая ветка
```

Эти переменные используются постоянно. Вот типичный пример — тегирование Docker-образа:

yamlcopy

```yaml
build-image:  script:    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

Образ будет называться `registry.gitlab.com/mycompany/myapp:a1b2c3d4`. Каждый коммит — уникальный тег. Всегда можно понять, из какого коммита собран образ.

### Приоритет переменных

Если одна и та же переменная задана в нескольких местах, приоритет (от высшего к низшему):

1. Trigger variables (при запуске пайплайна через API)
2. Project-level CI/CD variables (в UI проекта)
3. Group-level CI/CD variables (в UI группы)
4. Instance-level CI/CD variables (глобальные)
5. `.gitlab-ci.yml` variables
6. `.gitlab-ci.yml` job-level variables

То есть, переменная из UI проекта перекроет переменную из `.gitlab-ci.yml`. Это удобно — в файле задаёте дефолт, а в UI переопределяете для конкретного проекта.

## Environments

Environments — это привязка деплоя к конкретному окружению (staging, production). Не обязательная штука, но очень полезная для визуализации и контроля.

yamlcopy

```yaml
deploy-staging:  stage: deploy  script:    - ./deploy.sh staging  environment:    name: staging    url: https://staging.example.com deploy-prod:  stage: deploy  script:    - ./deploy.sh production  environment:    name: production    url: https://example.com  rules:    - if: $CI_COMMIT_BRANCH == "main"      when: manual
```

Что это даёт:

- В GitLab появляется раздел Operate → Environments, где видно: в staging задеплоена версия из коммита abc123, в production — из def456.
- Кнопка «Open» ведёт прямо на URL окружения.
- Видна история деплоев — кто, когда, какой коммит.
- Можно откатить деплой до предыдущей версии прямо из UI.

### Динамические environments

Для review apps — временных окружений под каждый merge request:

yamlcopy

```yaml
deploy-review:  stage: deploy  script:    - ./deploy-review.sh $CI_MERGE_REQUEST_IID  environment:    name: review/$CI_MERGE_REQUEST_IID    url: https://$CI_MERGE_REQUEST_IID.review.example.com    on_stop: stop-review  rules:    - if: $CI_PIPELINE_SOURCE == "merge_request_event" stop-review:  stage: deploy  script:    - ./destroy-review.sh $CI_MERGE_REQUEST_IID  environment:    name: review/$CI_MERGE_REQUEST_IID    action: stop  rules:    - if: $CI_PIPELINE_SOURCE == "merge_request_event"      when: manual
```

Открыли MR — автоматически поднялось тестовое окружение с этой веткой. Замёржили или закрыли MR — нажали кнопку stop, окружение удалилось. Тестировщик может проверить каждый MR на живом окружении, не мешая другим.
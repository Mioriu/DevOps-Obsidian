# GitLab CI

## .gitlab-ci.yml

Весь GitLab CI начинается с одного файла — `.gitlab-ci.yml`. Вы кладёте его в корень репозитория, и GitLab автоматически подхватывает его при каждом пуше. Запушили код — GitLab прочитал файл, построил пайплайн, запустил джобы. Без этого файла CI/CD не работает.

Файл написан на YAML

Минимальный рабочий пайплайн:

yamlcopy

```yaml
stages:  - build  - test  - deploy build-job:  stage: build  script:    - echo "Building the app..."    - npm install    - npm run build test-job:  stage: test  script:    - echo "Running tests..."    - npm test deploy-job:  stage: deploy  script:    - echo "Deploying..."    - ./deploy.sh
```

Вот и всё. Три стейджа, три джобы. Давайте разберём каждый элемент.

## Stages — этапы пайплайна

`stages` определяет порядок выполнения. Джобы внутри одного стейджа выполняются **параллельно**. Следующий стейдж начинается только когда все джобы предыдущего завершились успешно.

yamlcopy

```yaml
stages:  - build  - test  - deploy
```

![Stages в GitLab CI: build → test → deploy с параллельными джобами](https://offers.prostodevops.ru/diagrams/ci-cd/cicd-pipeline-stages.png)Stages в GitLab CI: build → test → deploy с параллельными джобами

Если хотя бы одна джоба в стейдже упала — следующий стейдж не запустится, пайплайн считается упавшим.

Если `stages` не указан, GitLab использует пять дефолтных: `.pre`, `build`, `test`, `deploy`, `.post`. Но лучше всегда прописывать явно — так понятнее, что происходит.

## Jobs — джобы

Джоба — это основная единица работы в пайплайне. Каждая джоба — это набор команд, которые выполняются на раннере. У джобы есть имя (произвольное), стейдж и скрипт:

yamlcopy

```yaml
unit-tests:              # имя джобы — произвольное  stage: test            # к какому стейджу относится  script:                # что выполнять    - pip install -r requirements.txt    - pytest tests/
```

Имя джобы можно задавать любое, но есть зарезервированные слова, которые нельзя использовать как имена: `stages`, `variables`, `include`, `image`, `services`, `before_script`, `after_script`, `pages`, `default`.

### image — в каком Docker-образе запускать

yamlcopy

```yaml
unit-tests:  stage: test  image: python:3.11-slim  script:    - pip install -r requirements.txt    - pytest tests/
```

Джоба запустится внутри контейнера `python:3.11-slim`. Это удобно — не нужно ничего ставить на раннер, всё описано в пайплайне. Для сборки Go-приложения — `image: golang:1.21`, для Node.js — `image: node:20-alpine`, и так далее.

`image` можно задать глобально для всех джоб:

yamlcopy

```yaml
default:  image: python:3.11-slim
```

И переопределить для конкретной джобы, если ей нужен другой образ.

## script, before_script, after_script

### script

Обязательное поле. Список команд, которые джоба выполняет:

yamlcopy

```yaml
build-job:  script:    - echo "Step 1"    - npm install    - npm run build
```

Команды выполняются последовательно. Если любая команда вернула ненулевой exit code — джоба падает. Как в обычном bash: команда упала → скрипт остановился.

Можно писать многострочные команды:

yamlcopy

```yaml
script:  - |    if [ "$CI_COMMIT_BRANCH" == "main" ]; then      echo "This is main branch"      ./deploy-prod.sh    else      echo "This is not main"      ./deploy-staging.sh    fi
```

### before_script

Выполняется **перед** `script`. Обычно тут ставят зависимости, логинятся в registry, подготавливают окружение:

yamlcopy

```yaml
unit-tests:  before_script:    - pip install -r requirements.txt  script:    - pytest tests/
```

Зачем выносить в `before_script`, а не писать всё в `script`? Во-первых, читаемость — сразу видно, где подготовка, а где основная работа. Во-вторых, `before_script` можно задать глобально через `default`:

yamlcopy

```yaml
default:  before_script:    - echo "This runs before every job" build-job:  script:    - echo "Building..." test-job:  script:    - echo "Testing..."
```

Обе джобы выполнят `before_script` перед своим `script`.

### after_script

Выполняется **после** `script`, причём **даже если script упал**. Используется для очистки — удалить временные файлы, отправить нотификацию, собрать логи:

yamlcopy

```yaml
integration-tests:  script:    - docker-compose up -d    - pytest tests/integration/  after_script:    - docker-compose down    - echo "Cleanup done"
```

`after_script` выполняется в отдельном shell-контексте. Это значит, что переменные окружения, заданные в `script`, в `after_script` будут недоступны.

## rules — когда запускать джобу

`rules` — это набор условий, определяющих, когда джоба должна запускаться. Это замена старых `only`/`except`, и GitLab рекомендует использовать именно `rules`.

yamlcopy

```yaml
deploy-prod:  stage: deploy  script:    - ./deploy.sh production  rules:    - if: $CI_COMMIT_BRANCH == "main"
```

Эта джоба запустится только при пуше в ветку `main`.

Правила проверяются сверху вниз. Первое совпавшее — применяется. Если ничего не совпало — джоба не запускается.

yamlcopy

```yaml
deploy:  stage: deploy  script:    - ./deploy.sh  rules:    # При пуше в main — деплоим автоматически    - if: $CI_COMMIT_BRANCH == "main"      when: on_success     # При пуше тега — деплоим вручную (нужно нажать кнопку)    - if: $CI_COMMIT_TAG      when: manual     # При merge request — не деплоим вообще    - if: $CI_PIPELINE_SOURCE == "merge_request_event"      when: never
```

### Условия в rules

yamlcopy

```yaml
rules:  # По ветке  - if: $CI_COMMIT_BRANCH == "main"  - if: $CI_COMMIT_BRANCH == "develop"   # По тегу  - if: $CI_COMMIT_TAG   # По источнику пайплайна  - if: $CI_PIPELINE_SOURCE == "merge_request_event"  - if: $CI_PIPELINE_SOURCE == "push"  - if: $CI_PIPELINE_SOURCE == "web"           # запущен руками из UI  - if: $CI_PIPELINE_SOURCE == "schedule"       # по расписанию   # По переменной  - if: $DEPLOY_ENV == "production"   # По изменённым файлам  - changes:      - backend/**/*      - requirements.txt
```

`changes` — полезная штука. Допустим, у вас монорепо с frontend и backend. Зачем запускать тесты бэкенда, если изменился только фронтенд?

yamlcopy

```yaml
backend-tests:  script:    - pytest  rules:    - changes:        - backend/**/*        - requirements.txt frontend-tests:  script:    - npm test  rules:    - changes:        - frontend/**/*        - package.json
```

### rules vs only/except

`only`/`except` — старый синтаксис, всё ещё работает:

yamlcopy

```yaml
# Старый способ — only/exceptdeploy:  script: ./deploy.sh  only:    - main  except:    - tags # Новый способ — rulesdeploy:  script: ./deploy.sh  rules:    - if: $CI_COMMIT_BRANCH == "main"
```

`rules` гибче и понятнее. `only`/`except` ограничены — нельзя комбинировать условия, нельзя задать `when` для отдельного правила. В новых пайплайнах используйте `rules`. Единственная причина знать `only`/`except` — вы встретите их в старых проектах.

Не смешивайте `rules` и `only`/`except` в одной джобе — GitLab выдаст ошибку.

## when — управление запуском

`when` определяет, при каком условии джоба запускается:

- **on_success** (дефолт) — запускается, если все джобы предыдущего стейджа прошли успешно.
- **on_failure** — запускается, только если что-то в предыдущем стейдже упало. Для нотификаций об ошибках.
- **always** — запускается всегда, независимо от результатов.
- **manual** — ждёт ручного нажатия кнопки в UI.
- **delayed** — запускается через определённое время.
- **never** — не запускается.

yamlcopy

```yaml
deploy-prod:  stage: deploy  script:    - ./deploy-prod.sh  rules:    - if: $CI_COMMIT_BRANCH == "main"      when: manual           # кнопка в UI, человек решает, когда деплоить notify-failure:  stage: deploy  script:    - ./send-alert.sh  when: on_failure            # отправить алерт, только если что-то упало
```

`when: manual` — это прямо очень частый паттерн для деплоя в прод. Сборка и тесты проходят автоматически, а деплой ждёт, пока кто-то нажмёт кнопку. Это даёт контроль: человек смотрит, что тесты прошли, и осознанно решает деплоить.

## allow_failure

По умолчанию, если джоба падает — весь пайплайн останавливается. `allow_failure: true` меняет это поведение: джоба может упасть, но пайплайн продолжит выполнение.

yamlcopy

```yaml
lint:  stage: test  script:    - npm run lint  allow_failure: true
```

Линтер упал — ок, отмечаем оранжевым предупреждением, но тесты и деплой продолжают работать.

Когда это полезно:

- Линтеры и стилистические проверки — хотите видеть результат, но не блокировать пайплайн.
- Экспериментальные тесты — нестабильные, пока не готовы быть обязательными.
- Ночные сканы безопасности — информируют, но не блокируют.

Для `when: manual` часто ставят `allow_failure: true`, чтобы пайплайн не висел в статусе «ожидает»:

yamlcopy

```yaml
deploy-prod:  stage: deploy  script:    - ./deploy-prod.sh  rules:    - if: $CI_COMMIT_BRANCH == "main"      when: manual      allow_failure: true     # пайплайн не будет "заблокирован" ожиданием деплоя
```

## needs — DAG (направленный ациклический граф)

По умолчанию джобы выполняются стейдж за стейджем. Но иногда это неэффективно. Допустим:

yamlcopy

```yaml
stages:  - build  - test  - deploy build-backend:  stage: build  script: make backend build-frontend:  stage: build  script: make frontend test-backend:  stage: test  script: make test-backend test-frontend:  stage: test  script: make test-frontend deploy:  stage: deploy  script: make deploy
```

По стандартной логике `test-backend` ждёт, пока завершится и `build-backend`, и `build-frontend`. Но зачем? Тестам бэкенда не нужен собранный фронтенд.

`needs` ломает стейджевую привязку и говорит: «запускайся сразу, как только вот эта конкретная джоба завершилась»:

yamlcopy

```yaml
test-backend:  stage: test  needs: ["build-backend"]     # не ждёт build-frontend  script: make test-backend test-frontend:  stage: test  needs: ["build-frontend"]    # не ждёт build-backend  script: make test-frontend deploy:  stage: deploy  needs: ["test-backend", "test-frontend"]  script: make deploy
```

![needs (DAG) vs обычные стейджи в GitLab CI](https://offers.prostodevops.ru/diagrams/ci-cd/cicd-needs-dag.png)needs (DAG) vs обычные стейджи в GitLab CI

С `needs` пайплайн работает быстрее, потому что джобы не ждут лишнего. На больших проектах с десятками джоб это может экономить минуты.

Ограничение: `needs` может ссылаться только на джобы из предыдущих или текущего стейджа. На джобы из будущих стейджей ссылаться нельзя.

## Полезные директивы

### extends — наследование

Чтобы не копипастить одни и те же настройки:

yamlcopy

```yaml
.base-test:                     # точка в начале — скрытая джоба, не выполняется  image: python:3.11-slim  before_script:    - pip install -r requirements.txt unit-tests:  extends: .base-test  script:    - pytest tests/unit/ integration-tests:  extends: .base-test  script:    - pytest tests/integration/
```

Обе джобы наследуют `image` и `before_script` от `.base-test`.

### include — подключение внешних файлов

Когда пайплайн разрастается, можно разбить его на файлы:

yamlcopy

```yaml
include:  - local: ci/build.yml  - local: ci/test.yml  - local: ci/deploy.yml   # Из другого репозитория  - project: 'devops/ci-templates'    ref: main    file: '/templates/docker-build.yml'   # По URL  - remote: 'https://example.com/ci-template.yml'
```

Это прямо необходимая вещь для больших проектов. Вместо одного файла на 500 строк — несколько маленьких, каждый отвечает за своё.

### tags — на каком раннере запускать

yamlcopy

```yaml
build-job:  tags:    - docker    - linux  script:    - make build
```
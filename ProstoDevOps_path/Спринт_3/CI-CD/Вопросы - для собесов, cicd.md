## Ключевые компоненты GitLab CI/CD?

GitLab CI/CD состоит из раннеров, пайплайнов, джобов и .gitlab-ci.yml файла (и на этом можно закончить)

**.gitlab-ci.yml** - это файл в корне репозитория где описан весь пайплайн. Как инструкция для Васи что делать по порядку - сначала собрать деталь, потом проверить качество, потом упаковать.

**Pipeline** - это набор джобов организованных по стейджам. Сначала стейдж сборки, потом тестирования, потом деплоя. Каждый стейдж должен успешно завершиться чтобы начался следующий.

**Jobs** - это отдельные задачи которые выполняются в пайплайне. Как отдельные Васи на конвейере - один собирает, другой тестирует, третий пакует. Джобы в одном стейдже выполняются параллельно, в разных стейджах - последовательно.

**GitLab Runner** - это агент который собственно выполняет джобы. Может быть shared (общий на весь GitLab), group (для группы проектов) или specific (для конкретного проекта). Раннер может быть где угодно - на вашем сервере, в докере, в кубере, в облаке.

**Artifacts** - файлы которые джоба создает и которые можно передать следующим джобам или скачать. Собрали бинарник в build джобе - передали его в test джобу через артефакты.

**Variables** - переменные для параметризации пайплайнов. Могут быть на уровне проекта, группы, инстанса. Плюс CI переменные которые GitLab сам создает (CI_COMMIT_SHA, CI_PIPELINE_ID и тд).

## Что такое before_script и after_script в GitLab CI/CD?

before_script - это команды которые выполняются перед основным script в каждой джобе. after_script - команды которые выполняются после script независимо от результата

**before_script** выполняется после клонирования репозитория но перед script. Типичные юзкейсы:

- Установка зависимостей
- Настройка окружения
- Логин в docker registry
- Экспорт переменных

**after_script** выполняется всегда, даже если джоба упала. Работает в отдельном shell контексте - не видит переменные из script. Типичные юзкейсы:

- Очистка временных файлов
- Отправка нотификаций
- Сбор отладочной информации при падении
- Логаут из систем

Важный момент - before_script и script работают в одном shell контексте, after_script - в отдельном. Если в before_script установили переменную - она будет доступна в script. В after_script - уже нет.

Если before_script упадет - джоба фейлится сразу, script не выполняется. Если упадет after_script - джоба все равно может быть успешной если script отработал нормально.

## У вас есть 5 проектов на одном языке программирования. Как организовать пайплайны, чтобы избежать дублирования конфигурации?

Нужно использовать include и extends для переиспользования конфигурации

**Создать общий репозиторий с шаблонами** и подключать через include:

yamlcopy

```yaml
include:  - project: 'devops/ci-templates'    file: '/templates/python.yml'
```

**Использовать extends** для наследования джобов:

yamlcopy

```yaml
.python_test_template:  image: python:3.9  script:    - pip install -r requirements.txt    - pytest test:  extends: .python_test_template  variables:    EXTRA_ARGS: "--coverage"
```

**Hidden jobs** (начинаются с точки) - это шаблоны которые не запускаются но могут быть унаследованы другими джобами.

**YAML anchors** для переиспользования кусков конфига:

yamlcopy

```yaml
.cache_template: &cache_config  cache:    key: ${CI_COMMIT_REF_SLUG}    paths:      - node_modules/
```

Best practice - создать отдельный репозиторий gitlab-ci-templates с шаблонами для разных языков и подключать нужные через include. Изменил шаблон в одном месте - обновилось во всех проектах.

## Как запускать тесты только при создании merge request?

Использовать rules или only с merge_requests

**Через rules:**

yamlcopy

```yaml
test:  script: npm test  rules:    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

**Через only (устаревший но работает):**

yamlcopy

```yaml
test:  script: npm test  only:    - merge_requests
```

**С дополнительными условиями:**

yamlcopy

```yaml
test:  script: npm test  rules:    - if: '$CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main"'      when: always    - if: '$CI_MERGE_REQUEST_LABELS =~ /skip-tests/'      when: never
```

## Если в before_script переопределить переменную, будет ли она доступна в script?

Да, будет доступна, потому что before_script и script выполняются в одном shell контексте

Порядок выполнения такой:

1. GitLab устанавливает CI переменные
2. Выполняется before_script
3. Выполняется script в том же shell
4. Выполняется after_script в НОВОМ shell

Пример:

yamlcopy

```yaml
job:  variables:    MY_VAR: "original"  before_script:    - export MY_VAR="modified"    - echo $MY_VAR  # выведет "modified"  script:    - echo $MY_VAR  # выведет "modified"  after_script:    - echo $MY_VAR  # выведет "original"
```

В after_script переменная вернется к оригинальному значению, потому что он запускается в новом shell контексте.

## Что такое кэширование в GitLab CI/CD?

Cache - это механизм сохранения файлов между запусками пайплайнов для ускорения сборки

Кеш позволяет не скачивать одни и те же зависимости каждый раз.

yamlcopy

```yaml
cache:  key: ${CI_COMMIT_REF_SLUG}  paths:    - node_modules/    - .npm/
```

**key** определяет уникальность кеша. Можно использовать:

- Статический ключ - один кеш для всех
- По ветке - `${CI_COMMIT_REF_SLUG}`
- По файлу - `${CI_COMMIT_REF_SLUG}-${CI_COMMIT_SHA}`
- По хешу файла - `cache:key:files: ["package-lock.json"]`

**policy** определяет поведение:

- `pull-push` (default) - скачать в начале, загрузить в конце
- `pull` - только скачать
- `push` - только загрузить

Важное отличие cache от artifacts - кеш не гарантирован. Раннер может его потерять, очистить, не найти. Поэтому кеш только для ускорения, а не для передачи критичных данных между джобами.

Типичные кейсы для кеша:

- Зависимости (node_modules, pip packages, maven .m2)
- Результаты компиляции которые редко меняются

## Как устроен Jenkins

Jenkins - это мастер-слейв архитектура где мастер управляет, а агенты выполняют работу

Jenkins состоит из нескольких компонентов:

**Jenkins Master (Controller)**. Хранит конфигурацию, планирует джобы, управляет агентами, предоставляет UI.

**Jenkins Agents (Nodes)** - выполняют билды. Могут быть на разных машинах, в контейнерах, в облаке. Агент получает задание от мастера, выполняет, отчитывается.

**Executors** - слоты для выполнения джобов на агенте. Если у агента 4 executor'а - может выполнять 4 джобы параллельно.

**Jobs/Projects** - конфигурация того что нужно сделать. Freestyle job, Pipeline, Multibranch Pipeline и тд.

**Plugins** - расширения функциональности. В Jenkins все через плагины - git, docker, kubernetes, notifications. Более 1500 плагинов.

**Workspace** - рабочая директория на агенте где выполняется джоба. Туда клонируется код, там происходит сборка.

## Как в Jenkins указать на каком воркере выполнять джобу

Через labels в declarative pipeline или node в scripted pipeline

**В Declarative Pipeline через agent:**

groovycopy

```groovy
pipeline {    agent { label 'linux && docker' }    // или конкретная нода    agent { node { label 'worker-1' } }}
```

**В Scripted Pipeline через node:**

groovycopy

```groovy
node('linux && docker') {    // код}
```

**Для отдельных стейджей:**

groovycopy

```groovy
pipeline {    agent none    stages {        stage('Build') {            agent { label 'maven' }            steps { ... }        }        stage('Test') {            agent { label 'selenium' }            steps { ... }        }    }}
```

Labels работают как теги. Агенту назначаете лейблы типа "linux", "docker", "gpu". В пайплайне указываете какие лейблы нужны. Jenkins найдет подходящего агента. Можно использовать логические операторы:

- `linux && docker` - нужны оба лейбла
- `linux || windows` - любой из них
- `!windows` - любой кроме windows

Если не указать агента - джоба запустится на мастере, что как бы плохая идея

## Что такое shared library

Shared Library - это способ переиспользовать Groovy код между пайплайнами в Jenkins

По сути это библиотека функций которую можно подключить к любому пайплайну. Вместо копипасты одного и того же кода в 100 Jenkinsfile'ов, выносите общую логику в библиотеку.

Структура shared library:

diagramcopy

```
(root)
├── src/           # Groovy классы
│   └── org/
│       └── foo/
│           └── Bar.groovy
├── vars/          # Глобальные переменные/функции
│   └── sayHello.groovy
└── resources/     # Ресурсные файлы
    └── scripts/
        └── build.sh
```

**vars/** - функции которые можно вызывать прямо из пайплайна:

groovycopy

```groovy
// vars/deployToK8s.groovydef call(String namespace, String image) {    sh "kubectl set image deployment/app app=${image} -n ${namespace}"}
```

Использование в пайплайне:

groovycopy

```groovy
@Library('my-shared-library') _pipeline {    stages {        stage('Deploy') {            steps {                deployToK8s('production', 'myapp:v1.0')            }        }    }}
```

Библиотеки можно подключать глобально (доступны всем), на уровне папки или в конкретном пайплайне. Версионировать через git теги или ветки.

Типичные юзкейсы - стандартные деплой процедуры, отправка уведомлений, работа с внешними системами, валидация, генерация версий.

## На каком языке пишется jenkinsfile

Jenkinsfile пишется на Groovy - это динамический язык для JVM

Есть два синтаксиса:

**Declarative Pipeline** - более новый и структурированный:

groovycopy

```groovy
pipeline {    agent any    stages {        stage('Build') {            steps {                sh 'make'            }        }    }}
```

**Scripted Pipeline** - старый, более гибкий, чистый Groovy:

groovycopy

```groovy
node {    stage('Build') {        sh 'make'    }}
```

Declarative проще и покрывает 95% кейсов. Структура жесткая - pipeline, agent, stages, steps. Легче читать и поддерживать. Есть валидация синтаксиса.

Scripted дает полную мощь Groovy - циклы, условия, try-catch, любые конструкции.

В Declarative можно использовать script блоки для Groovy кода:

groovycopy

```groovy
pipeline {    stages {        stage('Complex') {            steps {                script {                    // тут любой Groovy код                    if (env.BRANCH_NAME == 'main') {                        deploy()                    }                }            }        }    }}
```
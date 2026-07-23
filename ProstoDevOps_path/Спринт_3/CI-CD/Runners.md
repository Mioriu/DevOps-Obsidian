  

# GitLab Runners — типы, executor-ы, регистрация

## Что такое Runner

В предыдущем материале мы писали `.gitlab-ci.yml`, описывали джобы, скрипты. Но кто всё это выполняет? GitLab сам по себе — это просто интерфейс, он хранит код, показывает пайплайны, рисует графики. А физически команды выполняются на **раннерах**.

Runner — это агент (программа), который установлен на каком-то сервере или в кластере. Он подключается к GitLab, забирает джобы и выполняет их. Вот эта схема:

![GitLab Runner: GitLab → Runner → Окружение выполнения](https://offers.prostodevops.ru/diagrams/ci-cd/runner-flow.png)GitLab Runner: GitLab → Runner → Окружение выполнения

Runner постоянно опрашивает GitLab: «Есть для меня джобы?» Как только появляется подходящая — забирает и выполняет.

## Shared vs Specific (Group/Project)

### Shared Runners

Общие раннеры, доступные всем проектам на инстансе GitLab. Если вы используете gitlab.com — у вас из коробки есть shared раннеры, за них платить не нужно (есть лимит по минутам на бесплатном плане). Если у вас self-hosted GitLab — как у нас, то у нас есть свои раннеры.

Плюсы: не нужно ничего настраивать, работает сразу.

Минусы: раннер общий, значит очередь может быть длинной. Нет контроля над окружением — что установлено на раннере, то и используете. Не подходит, если нужен специфический софт или доступ к приватной сети.

### Specific Runners (Group / Project)

Раннер, привязанный к конкретному проекту или группе проектов. Вы сами ставите его на свой сервер, сами настраиваете, и он обслуживает только ваши проекты.

![Shared Runner vs Specific Runner](https://offers.prostodevops.ru/diagrams/ci-cd/runner-shared-vs-specific.png)Shared Runner vs Specific Runner

Когда нужен specific runner:

- Проекту нужен доступ к внутренней сети (например, деплой на серверы за VPN).
- Нужны специфические инструменты, которых нет на shared раннерах.
- Нужны GPU, много RAM, быстрые диски.
- Безопасность — не хотите, чтобы код крутился на чужих машинах.
- Нужна стабильная скорость без очередей.

В реальных проектах обычно комбинируют: shared раннеры для простых задач (линтер, тесты), specific — для сборки образов и деплоя.

## Executor — как Runner выполняет джобы

Runner — это обёртка. А то, как именно он выполняет джобы, определяет **executor**. Executor задаётся при регистрации раннера и определяет среду выполнения.

### Shell executor

Самый простой. Джоба выполняется прямо в shell на той машине, где установлен раннер. Как если бы вы зашли по SSH и руками набрали команды.

![Shell executor: джоба исполняется на хост-машине](https://offers.prostodevops.ru/diagrams/ci-cd/runner-shell-executor.png)Shell executor: джоба исполняется на хост-машине

Плюсы: просто, быстро (нет оверхеда на создание контейнеров), прямой доступ к системе.

Минусы: нет изоляции — одна джоба может насвинячить, и следующая получит грязное окружение. Нужно руками ставить все зависимости на сервер. Если два проекта требуют разные версии Node.js — проблема.

Когда использовать: деплой (нужен доступ к kubectl, ssh, внутренней сети), простые скрипты, ситуации, когда Docker недоступен.

### Docker executor

Каждая джоба запускается в отдельном Docker-контейнере. Образ берётся из поля `image` в `.gitlab-ci.yml`.

![Docker executor: каждая джоба — отдельный контейнер](https://offers.prostodevops.ru/diagrams/ci-cd/runner-docker-executor.png)Docker executor: каждая джоба — отдельный контейнер

Плюсы: полная изоляция, чистое окружение каждый раз, не нужно ставить зависимости на хост — всё в образе.

Минусы: оверхед на создание контейнера (обычно секунды), для сборки Docker-образов внутри контейнера нужен Docker-in-Docker или Kaniko.

Когда использовать: сборка, тесты, линтинг — практически для всего. Это дефолтный и самый популярный executor.

### Kubernetes executor

Каждая джоба запускается как pod в Kubernetes-кластере.

![Kubernetes executor: каждая джоба — Pod в кластере](https://offers.prostodevops.ru/diagrams/ci-cd/runner-k8s-executor.png)Kubernetes executor: каждая джоба — Pod в кластере

Плюсы: автоскейлинг — когда джоб много, Kubernetes создаёт больше подов. Когда мало — ресурсы освобождаются. Не нужно держать мощные серверы, которые простаивают 90% времени.

Минусы: сложнее в настройке, нужен работающий Kubernetes-кластер, отладка проблем сложнее.

Когда использовать: когда у вас уже есть Kubernetes, когда нагрузка на CI плавающая (то 5 джоб, то 50), когда нужен автоскейлинг.

## Регистрация раннера

Чтобы раннер появился и начал обслуживать проекты, его нужно установить и зарегистрировать.

### Установка

На Ubuntu/Debian:

bashcopy

```bash
# Добавляем репозиторий GitLabcurl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | bash # Ставимapt install gitlab-runner
```

После установки `gitlab-runner` работает как systemd-сервис:

systemctl status gitlab-runner

### Регистрация

Токен для регистрации берётся из GitLab: Settings → CI/CD → Runners.

gitlab-runner register

Интерактивно спросит:

1. **URL** — адрес вашего GitLab (`https://gitlab.com` или URL вашего self-hosted).
2. **Registration token** — токен из настроек проекта/группы.
3. **Description** — описание раннера (например, `prod-runner-01`).
4. **Tags** — теги (об этом ниже).
5. **Executor** — shell, docker, kubernetes и т.д.
6. Если Docker — **default image** (например, `alpine:latest`).

Можно и неинтерактивно, одной командой:

bashcopy

```bash
gitlab-runner register \  --non-interactive \  --url "https://gitlab.example.com" \  --token "$RUNNER_TOKEN" \  --executor "docker" \  --docker-image "alpine:latest" \  --description "docker-runner-01" \  --tag-list "docker,linux"
```

После регистрации раннер появится в GitLab в списке раннеров и начнёт забирать джобы.

### Конфиг раннера

После регистрации конфиг сохраняется в `/etc/gitlab-runner/config.toml`:

tomlcopy

```toml
[[runners]]  name = "docker-runner-01"  url = "https://gitlab.example.com"  token = "xxxxx"  executor = "docker"  [runners.docker]    image = "alpine:latest"    privileged = false    volumes = ["/cache"]
```

Тут можно тюнить параметры: лимиты CPU и памяти для контейнеров, привилегированный режим (нужен для Docker-in-Docker), маунты, pull policy для образов и т.д.

## Теги — как джоба находит свой раннер

Теги — это механизм маршрутизации джоб к раннерам. Вы при регистрации раннера задаёте теги, а в джобе указываете, какие теги нужны.

yamlcopy

```yaml
# .gitlab-ci.ymlbuild-job:  tags:    - docker    - linux  script:    - make build deploy-job:  tags:    - deploy    - production  script:    - ./deploy.sh
```

![GitLab tags: джоба ищет раннер с нужными тегами](https://offers.prostodevops.ru/diagrams/ci-cd/runner-tags.png)GitLab tags: джоба ищет раннер с нужными тегами

Джоба запустится только на раннере, у которого есть **все** указанные теги. Если подходящего раннера нет — джоба будет висеть в очереди, пока не появится.

Типичная схема тегов:

- `docker`, `linux` — для сборки и тестов в Docker-контейнерах.
- `deploy`, `staging` — раннер с доступом к staging-окружению.
- `deploy`, `production` — раннер с доступом к проду (отдельный, с усиленной безопасностью).
- `gpu` — раннер с GPU для ML-задач.

Если в джобе теги не указаны — она запустится на любом раннере, у которого включена опция «Run untagged jobs». Shared раннеры обычно эту опцию имеют, specific — не всегда.

## Как выбирается раннер для джобы

Алгоритм выбора:

![Алгоритм выбора Runner для джобы](https://offers.prostodevops.ru/diagrams/ci-cd/runner-selection-flow.png)Алгоритм выбора Runner для джобы

Если раннер занят — джоба ждёт. Количество одновременных джоб на раннере ограничено параметром `concurrent` в `config.toml`:

concurrent = 4    # раннер может выполнять до 4 джоб одновременно

По умолчанию `concurrent = 1` — одна джоба за раз. Для Docker executor можно спокойно поднимать, контейнеры изолированы. Для Shell executor — осторожнее, джобы будут конкурировать за ресурсы.
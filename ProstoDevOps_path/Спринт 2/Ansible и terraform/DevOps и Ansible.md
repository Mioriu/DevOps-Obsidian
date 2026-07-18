# Ansible в DevOps-практиках — Docker, Kubernetes, CI/CD и тестирование

Так, всю основную механику Ansible мы разобрали. Теперь давайте поговорим о том, как Ansible вписывается в реальный DevOps-стек. Потому что Ansible сам по себе — это хорошо, но в вакууме он не живёт. На реальной работе он крутится вместе с Docker, Kubernetes, GitLab CI, и всё это нужно уметь связывать. Плюс разберём, как тестировать Ansible-код

## Часть 1. Ansible + Docker

### Зачем Ansible, если есть Docker Compose

Справедливый вопрос. Docker Compose прекрасно поднимает контейнеры на одной машине. Но Ansible нужен, когда:

— Нужно подготовить сам хост — поставить Docker, настроить daemon.json, настроить сеть, файрвол, мониторинг. Docker Compose этого не умеет. — Нужно управлять контейнерами на нескольких серверах из одного места. — Нужно вписать Docker в общий флоу настройки инфраструктуры, где Ansible уже используется.

### Коллекция community.docker

Все модули для работы с Docker живут в коллекции `community.docker`. Ставим:

ansible-galaxy collection install community.docker

Или в `requirements.yml`:

yamlcopy

```yaml
collections:  - name: community.docker    version: "3.4.0"
```

### Провиженинг Docker-хоста

Первое, что нужно — поставить Docker на серверы. Можно через роль geerlingguy.docker, а можно написать своё:

yamlcopy

```yaml
---- name: Подготовка Docker-хостов  hosts: docker_hosts  become: true   tasks:    - name: Установить зависимости      apt:        name:          - apt-transport-https          - ca-certificates          - curl          - gnupg        state: present        update_cache: yes     - name: Добавить GPG-ключ Docker      apt_key:        url: https://download.docker.com/linux/ubuntu/gpg        state: present     - name: Добавить репозиторий Docker      apt_repository:        repo: "deb https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"        state: present     - name: Установить Docker      apt:        name:          - docker-ce          - docker-ce-cli          - containerd.io          - docker-compose-plugin        state: present     - name: Запустить Docker      service:        name: docker        state: started        enabled: yes     - name: Добавить пользователя deploy в группу docker      user:        name: deploy        groups: docker        append: yes
```

### Управление контейнерами

Модуль `docker_container` — создание и управление контейнерами:

yamlcopy

```yaml
    - name: Запустить контейнер с Redis      community.docker.docker_container:        name: redis        image: redis:7-alpine        state: started        restart_policy: unless-stopped        ports:          - "6379:6379"        volumes:          - redis_data:/data
```

Модуль `docker_image` — сборка и управление образами:

yamlcopy

```yaml
    - name: Собрать образ приложения      community.docker.docker_image:        name: myapp        tag: "{{ app_version }}"        source: build        build:          path: /opt/myapp          dockerfile: Dockerfile        push: false     - name: Запушить образ в реджистри      community.docker.docker_image:        name: "registry.company.local/myapp"        tag: "{{ app_version }}"        push: true        source: local
```

## Часть 2. Ansible + Kubernetes

### Когда это оправдано

Давайте сразу расставим точки. Если у вас есть Kubernetes — у вас наверняка есть kubectl, Helm, ArgoCD или FluxCD. И для деплоя приложений в кубер эти инструменты подходят лучше, чем Ansible. Это их прямая задача.

Ansible + Kubernetes оправдан в конкретных случаях:

— **Провиженинг самого кластера.** Подготовить ноды, поставить kubelet, kubeadm, развернуть кластер. Kubespray, кстати, это как раз Ansible-роли для разворачивания Kubernetes. — **Post-install настройка.** Кластер поднят, нужно накатить базовые вещи — CNI, Ingress Controller, cert-manager, мониторинг. Если у вас ещё нет ArgoCD — Ansible вполне подойдёт. — **Единый тулинг.** Вся инфраструктура управляется через Ansible, и вам проще добавить деплой в кубер туда же, чем заводить ещё один инструмент.

Если у вас уже настроен GitOps через ArgoCD или Flux — деплой приложений лучше делать через них. Ansible тут будет лишним

### Коллекция kubernetes.core

ansible-galaxy collection install kubernetes.core

Для работы нужен Python-пакет:

pip install kubernetes

### Модуль k8s — деплой манифестов

Основной модуль — `kubernetes.core.k8s`. Он умеет применять любые Kubernetes-манифесты:

yamlcopy

```yaml
---- name: Настройка Kubernetes  hosts: localhost  connection: local   tasks:    - name: Создать неймспейс      kubernetes.core.k8s:        state: present        definition:          apiVersion: v1          kind: Namespace          metadata:            name: myapp     - name: Задеплоить приложение      kubernetes.core.k8s:        state: present        src: manifests/deployment.yml        namespace: myapp     - name: Применить все манифесты из папки      kubernetes.core.k8s:        state: present        src: "{{ item }}"        namespace: myapp      with_fileglob:        - "manifests/*.yml"
```

Можно и через шаблоны — сгенерировать манифест из Jinja2 и применить:

yamlcopy

```yaml
    - name: Задеплоить из шаблона      kubernetes.core.k8s:        state: present        definition: "{{ lookup('template', 'deployment.yml.j2') | from_yaml }}"
```

### Helm через Ansible

Модуль `kubernetes.core.helm` управляет Helm-чартами:

yamlcopy

```yaml
    - name: Добавить репозиторий Prometheus      kubernetes.core.helm_repository:        name: prometheus-community        repo_url: https://prometheus-community.github.io/helm-charts     - name: Установить Prometheus через Helm      kubernetes.core.helm:        name: prometheus        chart_ref: prometheus-community/kube-prometheus-stack        release_namespace: monitoring        create_namespace: true        values:          grafana:            adminPassword: "{{ grafana_password }}"          prometheus:            retention: 30d
```

Это удобно, когда нужно развернуть инфраструктурные компоненты в кубере — мониторинг, ingress, cert-manager. Один плейбук — и кластер готов к работе.

## Часть 3. Ansible в CI/CD

Вот это прямо практическая тема, потому что запускать Ansible руками из консоли — это для разработки и отладки. На проде плейбуки должны запускаться автоматически, через пайплайн.

### GitLab CI — основной вариант

Типичный `.gitlab-ci.yml` для Ansible:

yamlcopy

```yaml
stages:  - lint  - check  - deploy variables:  ANSIBLE_FORCE_COLOR: "true"  ANSIBLE_HOST_KEY_CHECKING: "False" lint:  stage: lint  image: python:3.11-slim  before_script:    - pip install ansible-lint yamllint  script:    - yamllint .    - ansible-lint playbook.yml check:  stage: check  image: python:3.11-slim  before_script:    - pip install ansible    - ansible-galaxy install -r requirements.yml  script:    - echo "$VAULT_PASSWORD" > .vault_pass    - ansible-playbook playbook.yml --check --diff --vault-password-file .vault_pass    - rm -f .vault_pass  only:    - merge_requests deploy_staging:  stage: deploy  image: python:3.11-slim  before_script:    - pip install ansible    - ansible-galaxy install -r requirements.yml    - eval $(ssh-agent -s)    - echo "$SSH_PRIVATE_KEY" | ssh-add -  script:    - echo "$VAULT_PASSWORD" > .vault_pass    - ansible-playbook playbook.yml -i inventory/staging --vault-password-file .vault_pass    - rm -f .vault_pass  only:    - main  environment:    name: staging deploy_production:  stage: deploy  image: python:3.11-slim  before_script:    - pip install ansible    - ansible-galaxy install -r requirements.yml    - eval $(ssh-agent -s)    - echo "$SSH_PRIVATE_KEY" | ssh-add -  script:    - echo "$VAULT_PASSWORD_PROD" > .vault_pass    - ansible-playbook playbook.yml -i inventory/production --vault-password-file .vault_pass    - rm -f .vault_pass  only:    - main  when: manual  environment:    name: production
```

Разберём ключевые моменты.

**Линтинг** — первый этап, проверяем синтаксис и стиль. Если плейбук кривой — дальше не едем.

**Check** — сухой запуск на мерж-реквестах. Коллега видит в MR: "Ansible хочет поменять вот это и вот это." Можно ревьюить не только код плейбука, но и результат его работы.

**Деплой staging** — автоматически при пуше в main.

**Деплой production** — только руками (`when: manual`). Нажал кнопку — поехало. Это важно для прода, чтобы не было случайных выкаток.

### Хранение секретов в CI/CD

SSH-ключ и пароль от Vault хранятся в переменных GitLab CI (Settings → CI/CD → Variables):

— `SSH_PRIVATE_KEY` — приватный ключ для подключения к серверам. Тип: File или Variable, masked. — `VAULT_PASSWORD` — пароль от Ansible Vault. Тип: Variable, masked, protected. — `VAULT_PASSWORD_PROD` — отдельный пароль для продакшена.

Protected variables доступны только в protected-ветках. То есть, пароль от прода не утечёт через фиче-ветку.

### Jenkins

В Jenkins обычно делают через Jenkinsfile:

groovycopy

```groovy
pipeline {    agent { docker { image 'python:3.11-slim' } }     environment {        ANSIBLE_FORCE_COLOR = 'true'        ANSIBLE_HOST_KEY_CHECKING = 'False'    }     stages {        stage('Подготовка') {            steps {                sh 'pip install ansible ansible-lint'                sh 'ansible-galaxy install -r requirements.yml'            }        }         stage('Линтинг') {            steps {                sh 'ansible-lint playbook.yml'            }        }         stage('Деплой') {            steps {                withCredentials([                    sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'SSH_KEY'),                    string(credentialsId: 'vault-password', variable: 'VAULT_PASS')                ]) {                    sh '''                        eval $(ssh-agent -s)                        ssh-add $SSH_KEY                        echo "$VAULT_PASS" > .vault_pass                        ansible-playbook playbook.yml -i inventory/production --vault-password-file .vault_pass                        rm -f .vault_pass                    '''                }            }        }    }}
```

Секреты хранятся в Jenkins Credentials, доступ к ним — через `withCredentials`. Принцип тот же — SSH-ключ и Vault-пароль не лежат в коде, а подставляются в рантайме.

### Инвентори для пайплайнов

Типичная структура — отдельный инвентори на каждое окружение:

![Ansible inventory: staging vs production](https://offers.prostodevops.ru/diagrams/ansible/ansible-multi-env.png)Ansible inventory: staging vs production

В пайплайне просто указываете нужный инвентори:

bashcopy

```bash
# stagingansible-playbook playbook.yml -i inventory/staging # productionansible-playbook playbook.yml -i inventory/production
```

Плейбук один и тот же. Различия — только в инвентори и переменных. Это и есть Infrastructure as Code — одна кодовая база, разные окружения.

## Часть 4. Тестирование Ansible-кода

### yamllint — проверка YAML-синтаксиса

Самое базовое. Ловит кривые отступы, табы вместо пробелов, висячие пробелы:

bashcopy

```bash
pip install yamllintyamllint .
```

Конфиг `.yamllint.yml`:

yamlcopy

```yaml
---extends: defaultrules:  line-length:    max: 120  truthy:    allowed-values: ['true', 'false', 'yes', 'no']
```

### ansible-lint — проверка best practices

Ловит не синтаксические ошибки, а плохие практики: использование `shell` вместо модуля, отсутствие `name` у тасков, захардкоженные пароли и так далее:

bashcopy

```bash
pip install ansible-lintansible-lint playbook.yml
```

Пример вывода:

codecopy

```
playbook.yml:15: command-instead-of-module: Use apt module instead of shell for package managementplaybook.yml:23: name[missing]: All tasks should be namedplaybook.yml:31: yaml[truthy]: Truthy value should be one of [true, false, yes, no]
```

Каждое предупреждение — с объяснением, что не так и как исправить. Очень полезная штука, рекомендую запускать на каждый коммит.

Конфиг `.ansible-lint`:

yamlcopy

```yaml
---skip_list:  - role-name  # если имена ролей не по конвенции  - meta-no-info  # если meta/main.yml не заполнен
```

### Molecule — тестирование ролей

Molecule — это фреймворк для тестирования Ansible-ролей. Он создаёт тестовое окружение (контейнер или виртуалку), прогоняет роль внутри, проверяет результат и удаляет окружение.

pip install molecule molecule-plugins[docker]

Инициализация тестового сценария для роли:

bashcopy

```bash
cd roles/nginxmolecule init scenario --driver-name docker
```

Создаст структуру:

![Ansible molecule: converge.yml + molecule.yml + verify.yml](https://offers.prostodevops.ru/diagrams/ansible/ansible-molecule.png)Ansible molecule: converge.yml + molecule.yml + verify.yml

`molecule.yml`:

yamlcopy

```yaml
---driver:  name: dockerplatforms:  - name: ubuntu-test    image: ubuntu:22.04    pre_build_image: true    command: /sbin/init    privileged: trueprovisioner:  name: ansibleverifier:  name: ansible
```

`converge.yml`:

yamlcopy

```yaml
---- name: Тест роли nginx  hosts: all  become: true  roles:    - role: nginx
```

`verify.yml`:

yamlcopy

```yaml
---- name: Проверка после применения  hosts: all   tasks:    - name: Проверить, что nginx установлен      command: nginx -v      changed_when: false     - name: Проверить, что nginx запущен      service_facts:     - name: Убедиться, что nginx активен      assert:        that:          - "'nginx.service' in ansible_facts.services"          - "ansible_facts.services['nginx.service'].state == 'running'"     - name: Проверить, что порт 80 слушается      wait_for:        port: 80        timeout: 5
```

Запуск:

bashcopy

```bash
# полный цикл: создать → применить → проверить → удалитьmolecule test # или по шагамmolecule create     # создать контейнерmolecule converge   # применить рольmolecule verify     # проверить результатmolecule destroy    # удалить контейнерmolecule login      # зайти в контейнер для дебага
```

`molecule test` — это полный цикл. Создаёт контейнер, прогоняет роль, запускает проверки, удаляет контейнер. Если хоть один этап упал — тест провален.

`molecule converge` + `molecule login` — это для отладки. Применили роль, зашли в контейнер, посмотрели руками, что получилось.

### Testinfra и Goss — проверка результата

Если `verify.yml` на Ansible не хватает, можно использовать Testinfra — Python-фреймворк для тестирования инфраструктуры:

pythoncopy

```python
# molecule/default/tests/test_default.py def test_nginx_installed(host):    nginx = host.package("nginx")    assert nginx.is_installed def test_nginx_running(host):    nginx = host.service("nginx")    assert nginx.is_running    assert nginx.is_enabled def test_nginx_listening(host):    socket = host.socket("tcp://0.0.0.0:80")    assert socket.is_listening def test_config_exists(host):    config = host.file("/etc/nginx/nginx.conf")    assert config.exists    assert config.user == "root"    assert config.contains("worker_processes")
```

Читается как обычный текст: nginx установлен? Запущен? Порт слушает? Конфиг на месте? Всё это Testinfra проверяет внутри контейнера, который создал Molecule.

Goss — альтернатива на Go, работает через YAML:

yamlcopy

```yaml
# goss.yamlservice:  nginx:    enabled: true    running: true port:  tcp:80:    listening: true file:  /etc/nginx/nginx.conf:    exists: true    owner: root
```

Goss быстрее, Testinfra гибче. Для большинства случаев хватает обоих.
# Helm: charts, values, релизы

Тип: Урок · Статус: вычитано Обновлено: 07.06.2026, 15:46

## Манифесты размножаются

Посчитаем, во что выросло одно приложение за два модуля. Deployment с пробами и ресурсами, Service, Ingress с TLS, ConfigMap, Secret, PVC – шесть манифестов на сотни строк, и это один стенд. Для staging и production нужны почти такие же комплекты с точечными различиями: другой образ, другое число реплик, другой хост в Ingress, другие ресурсы. Копировать директорию манифестов под каждое окружение – путь известный, копии немедленно начинают разъезжаться. Болезнь та же, что была с плейбуками до ролей и с Terraform-кодом до модулей, и лечится она тем же – шаблонизацией с параметрами.

Вторая половина проблемы – чужой софт. За модуль в кластере понадобились Ingress Controller, cert-manager, metrics-server, дальше понадобится мониторинг. Каждый из них – десятки связанных манифестов с сотнями параметров, и собирать их руками из документации никто не предлагает. Нужен способ ставить готовое одной командой.

Обе задачи решает **Helm** – пакетный менеджер Kubernetes. Пакет в его терминах называется **чартом** (chart) – это манифесты, превращенные в шаблоны, плюс файл параметров. Установленный в кластер чарт с конкретными параметрами называется **релизом** (release). По месту в экосистеме это apt из Linux, только пакеты здесь разворачиваются в объекты кластера.

![Helm: chart плюс values рендерятся в release как обычные объекты кластера](https://prostodevops.ru/api/uploads/lessons/helm-charts-values-relizy-01.webp)Helm: chart плюс values рендерятся в release как обычные объекты кластера

Ставится Helm как обычный бинарник на вашу машину, рядом с kubectl, и работает через тот же kubeconfig:

helm version

## Установка чужих чартов

Начнем со стороны потребителя. Чарты распространяются через репозитории, и работа с ними устроена привычно по пакетным менеджерам:

Bash

Копировать

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo postgresql
```

Bitnami – один из крупнейших публичных каталогов, у большинства проектов есть и собственные репозитории. Установка:

helm install my-db bitnami/postgresql

Копировать

```
NAME: my-db
NAMESPACE: default
STATUS: deployed
REVISION: 1
```

Здесь `my-db` – имя релиза, `bitnami/postgresql` – чарт. За одну команду Helm отрендерил шаблоны чарта в манифесты и применил их – в кластере появились StatefulSet, сервисы, секреты, все объекты из прошлых уроков. Никакой магии поверх Kubernetes нет, `kubectl get all` показывает обычные объекты, и вся выученная отладка работает с ними как раньше. Установленное смотрится так:

Bash

Копировать

```bash
helm list
helm status my-db
```

Параметры чарта живут в его values – у серьезных чартов их сотни, от версии образа до настроек репликации. Посмотреть доступное:

helm show values bitnami/postgresql

Переопределяются параметры двумя способами:

Bash

Копировать

```bash
helm install my-db bitnami/postgresql --set auth.postgresPassword=s3cr3t
helm install my-db bitnami/postgresql -f my-values.yaml
```

Флаг `--set` годится для разовых экспериментов. Для жизни значения складываются в свой файл `my-values.yaml` – он содержит только отличия от умолчаний, лежит в Git и проходит ревью, как любая конфигурация. И знакомое правило фиксации версий действует и здесь:

helm install my-db bitnami/postgresql --version 16.2.1 -f my-values.yaml

Чарт без версии в разные дни поставит разное.

## Релизы и ревизии

Обновление релиза – изменение values или версии чарта:

helm upgrade my-db bitnami/postgresql -f my-values.yaml

Каждый `upgrade` создает новую ревизию релиза, и история ведется автоматически:

![Релизы и ревизии: история upgrade и откат helm rollback](https://prostodevops.ru/api/uploads/lessons/helm-charts-values-relizy-03.webp)Релизы и ревизии: история upgrade и откат helm rollback

Копировать

```
REVISION  STATUS      CHART              DESCRIPTION
1         superseded  postgresql-16.2.1  Install complete
2         deployed    postgresql-16.2.1  Upgrade complete
```

Откат на любую ревизию – одна команда, прямой родственник `rollout undo`:

helm rollback my-db 1

Удаляется релиз вместе со всеми своими объектами:

helm uninstall my-db

Это сильная сторона релизной модели – Helm помнит, какие объекты принадлежат релизу, и убирает их все. Руками пришлось бы выискивать каждый Deployment, сервис и секрет по отдельности. Сама история релизов хранится в секретах того же namespace, отдельной базы у Helm нет.

## Устройство чарта

Теперь сторона производителя – свой чарт для своего приложения. Скелет создается командой:

![Устройство чарта и три источника значений: .Values, .Release, .Chart](https://prostodevops.ru/api/uploads/lessons/helm-charts-values-relizy-02.webp)Устройство чарта и три источника значений: .Values, .Release, .Chart

Копировать

```
myapp/
├── Chart.yaml          # метаданные чарта
├── values.yaml         # параметры по умолчанию
├── templates/          # шаблоны манифестов
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl
└── charts/             # вложенные чарты-зависимости
```

**Chart.yaml** – паспорт пакета:

YAML

Копировать

```yaml
apiVersion: v2
name: myapp
version: 1.2.0
appVersion: "1.5.3"
```

Поля `version` и `appVersion` различаются. Первое – версия самого чарта, она растет при изменении шаблонов. Второе – версия упакованного приложения, информационное поле. Чарт может обновиться без смены приложения и наоборот.

**values.yaml** – параметры по умолчанию:

YAML

Копировать

```yaml
replicaCount: 2
image:
  repository: registry.example.com/myapp
  tag: "1.5.3"
ingress:
  host: app.example.com
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

Назначение файла то же, что у `defaults` в роли Ansible. Чарт работает из коробки с разумными значениями, а пользователь переопределяет только нужное своим файлом.

**templates/** – манифесты, в которые вставлены подстановки:

YAML

Копировать

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

Синтаксис – Go-шаблоны, по духу близкие к Jinja2 из Ansible: те же двойные скобки, фильтры через `|`, условия и циклы. Источников значений три. `.Values` – все из values.yaml с учетом переопределений. `.Release` – данные релиза, и подстановка `.Release.Name` в имена объектов позволяет ставить один чарт несколько раз без конфликтов имен. `.Chart` – данные из Chart.yaml. Конструкция `toYaml ... | nindent 12` вставляет целый блок значений с нужным отступом – типовой прием для секций вроде `resources`, чтобы их структура задавалась прямо в values. Файл `_helpers.tpl` хранит именованные куски шаблонов для переиспользования, в сгенерированном скелете там собраны стандартные метки.

Проверяется рендер без касания кластера:

Bash

Копировать

```bash
helm template myapp ./myapp -f values-prod.yaml
helm lint ./myapp
helm install myapp ./myapp --dry-run --debug
```

`template` печатает готовые манифесты – что именно уедет в кластер, видно глазами до установки. `lint` ловит ошибки структуры чарта.

## Окружения

Возвращаемся к проблеме из начала урока. С чартом комплект манифестов существует в одном экземпляре – как шаблоны. А различия стендов сводятся в маленькие файлы значений:

![Один чарт, разные окружения через values-staging и values-prod](https://prostodevops.ru/api/uploads/lessons/helm-charts-values-relizy-04.webp)Один чарт, разные окружения через values-staging и values-prod

YAML

Копировать

```yaml
# values-prod.yaml
replicaCount: 4
image:
  tag: "1.5.3"
ingress:
  host: app.example.com
resources:
  requests:
    cpu: 500m
    memory: 512Mi
```

Bash

Копировать

```bash
helm install myapp-staging ./myapp -f values-staging.yaml -n staging
helm install myapp ./myapp -f values-prod.yaml -n production
```

Шаблоны общие, разница окружений видна целиком в двух коротких файлах – конструкция, знакомая по tfvars в Terraform и group_vars в Ansible. Правка шаблона автоматически касается всех стендов при следующем обновлении, копии разъезжаться перестали.

В CI деплой обычно записывается одной командой:

helm upgrade --install myapp ./myapp -f values-prod.yaml --set image.tag=$CI_COMMIT_TAG

Флаг `--install` делает команду идемпотентной – релиза нет, он установится, есть – обновится. Один и тот же шаг пайплайна работает и для первого деплоя, и для сотого, а тег образа приезжает из переменной CI поверх файла значений.

## Границы инструмента

Пара честных замечаний по опыту эксплуатации. Go-шаблоны при разрастании чарта читаются тяжело – манифест, наполовину состоящий из `{{- if }}` и `nindent`, далек от YAML, который изучался весь модуль, и отлаживается через многократный `helm template`. Поэтому свои чарты стоит держать простыми: параметризовать то, что реально различается между стендами, и не превращать каждый байт манифеста в переменную. Для случаев, где шаблонизация кажется избыточной, в экосистеме есть альтернативный подход kustomize – наложение точечных правок поверх готовых манифестов, он встроен в kubectl. На рынке тем не менее стандарт де-факто – Helm: чужой софт распространяется чартами, вакансии ожидают его, и команда `helm upgrade --install` из CI – самый распространенный вид продового деплоя в Kubernetes.

## Шпаргалка

Bash

Копировать

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami   # подключить репозиторий
helm search repo postgresql                # найти чарт
helm show values bitnami/postgresql        # параметры чарта
helm install NAME CHART -f values.yaml     # установить
helm list                                  # релизы в namespace
helm upgrade NAME CHART -f values.yaml     # обновить
helm upgrade --install NAME CHART          # установить или обновить (CI)
helm history NAME                          # ревизии релиза
helm rollback NAME 1                       # откат на ревизию
helm uninstall NAME                        # удалить релиз с объектами
helm create myapp                          # скелет своего чарта
helm template myapp ./myapp -f values.yaml # отрендерить локально
helm lint ./myapp                          # проверить чарт
```
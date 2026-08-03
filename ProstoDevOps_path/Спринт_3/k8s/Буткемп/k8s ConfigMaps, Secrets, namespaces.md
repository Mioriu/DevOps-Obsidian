
## Конфиг отдельно от образа

Принцип, заведенный еще в модуле про Docker, никуда не делся. Образ один на все стенды, а различия между стендами приходят снаружи через переменные окружения. В `docker run` это решал флаг `-e`, в Compose – секция `environment` и `env_file`. В манифесте пода переменные тоже задаются, прямо в описании контейнера:

YAML

Копировать

```yaml
spec:
  containers:
    - name: web
      image: myapp:1.4
      env:
        - name: APP_MODE
          value: "production"
        - name: LOG_LEVEL
          value: "info"
```

Для пары переменных это нормально. Дальше начинаются неудобства. Конфигурация размазывается по манифестам приложений – у пяти Deployment, которым нужен адрес одной базы, он прописан пять раз. Смена значения означает правку манифестов приложений, хотя само приложение не менялось. А для конфигов-файлов вроде `nginx.conf` секция `env` вообще не подходит.

Kubernetes выносит конфигурацию в отдельные объекты. Несекретное – в ConfigMap, секретное – в Secret, и оба подключаются к подам по ссылке.

## ConfigMap

**ConfigMap** – объект с конфигурационными данными в виде пар ключ-значение:

YAML

Копировать

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
  nginx.conf: |
    server {
        listen 80;
        root /usr/share/nginx/html;
    }
```

Вся содержательная часть лежит в `data`. Ключи бывают двух стилей. Короткие значения вроде `APP_MODE` предназначены стать переменными окружения. А ключ с многострочным значением через `|` – это целый файл, `nginx.conf` здесь хранится внутри объекта целиком. ConfigMap всеяден, ему безразлично, что лежит в значениях.

Кроме манифеста, ConfigMap создается императивно из готовых значений и файлов:

Bash

Копировать

```bash
kubectl create configmap web-config --from-literal=APP_MODE=production
kubectl create configmap nginx-config --from-file=nginx.conf
```

Второй вариант кладет файл с диска в ConfigMap под ключом с именем файла. И здесь снова полезен трюк из урока про манифесты – добавить `--dry-run=client -o yaml`, получить манифест и положить его в Git, вместо того чтобы создавать объект мимо файлов.

## Подключение к поду

Сам по себе ConfigMap просто лежит в кластере.

![ConfigMap и Secret подключаются к поду: одна переменная, все ключи разом или монтирование файлами](https://prostodevops.ru/api/uploads/lessons/configmaps-secrets-namespaces-01.webp)ConfigMap и Secret подключаются к поду: одна переменная, все ключи разом или монтирование файлами

Работать он начинает, когда под на него ссылается, и способов три.

Первый – забрать одно значение в одну переменную:

YAML

Копировать

```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: web-config
        key: LOG_LEVEL
```

Второй – забрать все ключи разом, каждый станет переменной окружения:

YAML

Копировать

```yaml
envFrom:
  - configMapRef:
      name: web-config
```

Третий – смонтировать ConfigMap как файлы. Каждый ключ превращается в файл в указанной директории:

YAML

Копировать

```yaml
spec:
  containers:
    - name: web
      image: nginx:1.27
      volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: config
      configMap:
        name: nginx-config
```

В `volumes` объявляется том из ConfigMap, в `volumeMounts` он подключается в контейнер по пути. После запуска в `/etc/nginx/conf.d` лежит файл `nginx.conf` с содержимым из объекта. Это и есть ответ на конфиги-файлы – они хранятся в кластере и доезжают до контейнера без запекания в образ. По смыслу это место `template` из Ansible, разложить конфиг по серверам, только сервером теперь выступает под.

## Обновление конфига

Здесь есть проблема. Поменяли значение в ConfigMap, применили – а приложение работает по-старому.

Переменные окружения выдаются процессу при старте, ровно как в Linux. Под запущен – его окружение зафиксировано, и никакая правка ConfigMap его не изменит. Файлы через том обновляются внутри работающего пода, но с задержкой до минуты, и приложение должно само уметь перечитывать конфиг, что умеют немногие.

Рабочий прием уже знаком по уроку про Deployment:

Bash

Копировать

```bash
kubectl apply -f configmap.yaml
kubectl rollout restart deployment/web
```

Правка конфига, затем перекат подов. Новые поды стартуют с новым окружением и новыми файлами, старые гасятся – без даунтайма, обычным rolling update.

## Secrets

**Secret** – тот же ConfigMap по устройству, но для чувствительных данных. Пароли, токены, ключи API, TLS-сертификаты:

Bash

Копировать

```bash
kubectl create secret generic db-secret \
  --from-literal=POSTGRES_USER=app \
  --from-literal=POSTGRES_PASSWORD=s3cr3t-pAssw0rd
```

Посмотрим, что получилось:

kubectl get secret db-secret -o yaml

YAML

Копировать

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  POSTGRES_USER: YXBw
  POSTGRES_PASSWORD: czNjcjN0LXBBc3N3MHJk
```

Значения выглядят зашифрованными, и вот тут главное, что надо понять про секреты в Kubernetes. **Это base64, и base64 не является шифрованием.** Это кодировка для переноса любых байтов текстом, снимается она одной командой:

echo 'czNjcjN0LXBBc3N3MHJk' | base64 -d

s3cr3t-pAssw0rd

Никакого пароля, никакого ключа – любой, кто прочитал объект, прочитал секрет. Зачем тогда отдельный тип? У разделения есть реальные причины. Права в кластере раздаются по типам объектов, и доступ на чтение ConfigMap выдается широко, а на чтение Secret – узко. Вывод `describe secret` скрывает значения, показывая только размеры, и инструменты экосистемы относятся к типу бережнее. Для etcd включается шифрование секретов при хранении. Но все это работает, только если секреты лежат в Secret, а не в ConfigMap рядом – поэтому правило жесткое, чувствительное значение никогда не попадает в ConfigMap.

В манифесте секрета кодировать значения руками не нужно, для этого есть поле `stringData`:

YAML

Копировать

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  POSTGRES_USER: app
  POSTGRES_PASSWORD: s3cr3t-pAssw0rd
```

Значения пишутся как есть, кластер сам перекодирует их в `data` при сохранении.

Подключение к подам полностью повторяет ConfigMap – `secretKeyRef` для одной переменной, `envFrom` с `secretRef` для всех, том для файлов:

YAML

Копировать

```yaml
envFrom:
  - configMapRef:
      name: web-config
  - secretRef:
      name: db-secret
```

## Секреты и Git

Манифест выше создает проблему, знакомую по модулю Ansible. Файл с `stringData` и реальным паролем в репозитории – это пароль в Git, со всей историей коммитов и ботами-сканерами. Base64 в `data` от ботов не спасает, его декодируют так же автоматически.

Дисциплина переносится из урока про vault без изменений. В Git живут манифесты приложений и ConfigMap, а манифесты секретов с реальными значениями – нет. Сами секреты доставляются в кластер другим каналом – создаются из переменных CI при деплое, либо через специальные инструменты, которые хранят в Git зашифрованную форму или подтягивают значения из внешнего хранилища. Таких инструментов несколько, Sealed Secrets и External Secrets Operator – самые ходовые, и при встрече с ними на работе вы узнаете ровно ту же идею, что была в ansible-vault.

## Namespaces

Третья тема урока отвечает на вопрос, который уже всплывал дважды. В выводах команд мелькали `kube-system` и `default` – это **namespaces**, пространства имен, способ разделить один кластер на изолированные части.

kubectl get namespaces

Копировать

```
NAME              STATUS   AGE
default           Active   12d
kube-system       Active   12d
kube-public       Active   12d
kube-node-lease   Active   12d
```

Служебные компоненты живут в `kube-system`, а все, что мы создавали до сих пор, попадало в `default`, потому что другого не указывали. Создаются пространства одной командой или манифестом:

kubectl create namespace staging

Дальше пространство указывается флагом `-n` в командах или полем в манифесте:

Bash

Копировать

```bash
kubectl get pods -n staging
kubectl apply -f deployment.yaml -n staging
```

YAML

Копировать

```yaml
metadata:
  name: web
  namespace: staging
```

Имена объектов уникальны в пределах пространства, и это главное практическое свойство.

![Namespaces делят кластер на изолированные окружения staging/production со своими объектами; обращение через DNS](https://prostodevops.ru/api/uploads/lessons/configmaps-secrets-namespaces-02.webp)Namespaces делят кластер на изолированные окружения staging/production со своими объектами; обращение через DNS

Deployment `web` может существовать и в `staging`, и в `production` одного кластера, не мешая друг другу, – с разными ConfigMap, разными секретами и разным числом реплик. Типовые применения отсюда очевидны: окружения, команды, отдельные проекты на общем кластере.

С пространствами связана и полная форма DNS-имени из урока про сервисы. `web.staging.svc.cluster.local` – сервис `web` в пространстве `staging`. Внутри одного пространства хватает короткого имени `web`, а через границу обращаются по форме `имя.пространство` – например, `web.staging`. ConfigMap и Secret при этом через границу не видны вообще, под подключает только объекты своего пространства, и это осознанное ограничение – у каждого окружения своя конфигурация и свои секреты.

Постоянно писать `-n staging` надоедает быстро, рабочее пространство переключается в контексте:

kubectl config set-context --current --namespace=staging

После этого все команды по умолчанию работают со `staging`. Здесь же повторю привычку из урока про kubectl – `kubectl config current-context` перед серьезными действиями, теперь вместе с пространством. Команда в чужом namespace – родная сестра команды на чужом кластере.

Не все объекты живут в пространствах – ноды и некоторые кластерные ресурсы общие на весь кластер, колонка `NAMESPACED` в `kubectl api-resources` показывает, что есть что. И namespace сам по себе не граница безопасности – сеть между пространствами по умолчанию открыта, под из `staging` дотянется до сервиса в `production` по полному имени. Пространство – это граница организации и видимости, а точкой приложения для прав доступа и сетевых ограничений оно становится с помощью отдельных механизмов, RBAC и сетевых политик.

## Все вместе

Сквозной пример собирает урок целиком – окружение, конфиг, секрет и приложение, которое их использует:

YAML

Копировать

```yaml
# staging.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
---

apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
  namespace: staging
data:
  APP_MODE: "staging"
  LOG_LEVEL: "debug"
---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: myapp:1.4
          envFrom:
            - configMapRef:
                name: web-config
            - secretRef:
                name: db-secret
```

Секрета в файле нет намеренно, он создается отдельной командой до применения:

Bash

Копировать

```bash
kubectl create secret generic db-secret -n staging \
  --from-literal=POSTGRES_PASSWORD=s3cr3t-pAssw0rd
kubectl apply -f staging.yaml
```

Приложение получает окружение из двух источников, не зная о них ничего. Тот же образ в `production` поедет с другим ConfigMap и другим секретом, без единого изменения в самом приложении – конфигурация окончательно отделена и от образа, и от манифеста приложения.

## Шпаргалка

Bash

Копировать

```bash
kubectl create configmap NAME --from-literal=KEY=value    # из значения
kubectl create configmap NAME --from-file=nginx.conf      # из файла
kubectl create secret generic NAME --from-literal=KEY=value
kubectl get configmaps                                    # список
kubectl get secret NAME -o yaml                           # секрет целиком (base64!)
kubectl rollout restart deployment/NAME                   # перекатить после правки конфига
kubectl get namespaces                                    # пространства имен
kubectl create namespace staging
kubectl get pods -n staging                               # команды в пространстве
kubectl config set-context --current --namespace=staging  # рабочее пространство
```
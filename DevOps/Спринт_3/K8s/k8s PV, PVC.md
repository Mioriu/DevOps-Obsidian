# PersistentVolume и PersistentVolumeClaim в Kubernetes

## Где это может пригодиться:

- Развернуть базу данных PostgreSQL, которая должна сохранять данные после перезапуска пода
    
- Настроить хранилище для логов приложения, чтобы они не терялись при обновлении
    
- Организовать общую директорию для нескольких подов, где они смогут обмениваться файлами
    
- Перенести под на другой узел кластера без потери данных
    
- Создать резервные копии данных из контейнеров
    
- Масштабировать stateful приложение (Redis, MongoDB) с гарантией сохранности данных
    
- Дать разработчикам возможность запрашивать хранилище без знания деталей инфраструктуры
    

На собеседованиях могут спросить:

- "В чём разница между PV и PVC?"
    
- "Какие типы volumes поддерживает Kubernetes?"
    
- "Как работает Dynamic Provisioning?"
    
- "Что будет с PV после удаления PVC?"
    
- "Как настроить постоянное хранилище для базы данных в Kubernetes?"
    

## Давай разберемся.

## 1. Что такое PersistentVolume и PersistentVolumeClaim?

### Определение

**PersistentVolume (PV)** — это ресурс кластера Kubernetes, а именно - **физическое** хранилище (диск, NFS, облачное хранилище).  
**PersistentVolumeClaim (PVC)** — это **запрос на выделение хранилища**.  
**Как это работает:**

1. **Создаём PV** — ресурсы хранилища в кластере
    
2. **Далее создаём PVC** — указываем размер и требования
    
3. **Kubernetes связывает PVC с подходящим PV** — выделяет хранилище бедолаге
    
4. **Под использует PVC** — бедолага использует хранилище
    

**Зачем нужно разделение на PV и PVC:**

- ✅ **Абстракция** — разработчики не знают детали инфраструктуры (где физически лежат данные)
    
- ✅ **Разделение ответственности** — админы управляют хранилищем, разработчики его запрашивают
    
- ✅ **Переносимость** — можно менять бэкендовые хранилища не трогая приложения
    
- ✅ **Динамическое выделение** — можно автоматически создавать PV по требованию
    
- ✅ **Независимый жизненный цикл** — данные живут дольше чем поды
    

### Простой пример

**Проблема без PV/PVC:**

javascriptcopy

```javascript
# Pod без постоянного хранилищаapiVersion: v1kind: Podmetadata:  name: databasespec:  containers:  - name: postgres    image: postgres:13    # Данные хранятся внутри контейнера    # При перезапуске пода - всё потеряно
```

**Проблемы:**

- 😱 Все данные удаляются при перезапуске пода
    
- 😱 Невозможно мигрировать под на другую ноду с сохранением данных
    
- 😱 Нет резервных копий
    
- 😱 Невозможно масштабировать
    

**Решение с PV/PVC:**

javascriptcopy

```javascript
# 1. Создаём PVC - запрос на хранилищеapiVersion: v1kind: PersistentVolumeClaimmetadata:  name: postgres-pvcspec:  accessModes:    - ReadWriteOnce  # Может монтироваться в один под  resources:    requests:      storage: 10Gi  # Просим 10 ГБ ---# 2. Используем PVC в подеapiVersion: v1kind: Podmetadata:  name: databasespec:  containers:  - name: postgres    image: postgres:13    volumeMounts:    - name: postgres-storage      mountPath: /var/lib/postgresql/data  # Куда монтируем внутри контейнера  volumes:  - name: postgres-storage    persistentVolumeClaim:      claimName: postgres-pvc  # Ссылка на наш PVC
```

**Преимущества:**

- ✅ Данные сохраняются при перезапуске пода
    
- ✅ Можно мигрировать под с данными
    
- ✅ Можно создавать бэкапы
    
- ✅ Данные живут независимо от жизненного цикла пода
    

## 2. Базовые концепции

### 2.1. Жизненный цикл PV и PVC

**Простыми словами:** PV и PVC проходят через несколько состояний от создания до удаления.  
**Статусы PersistentVolume:**

|Статус|Описание|Что это значит|
|---|---|---|
|`Available`|Доступен|PV создан, но не привязан к PVC|
|`Bound`|Привязан|PV привязан к конкретному PVC|
|`Released`|Освобождён|PVC удалён, но PV ещё не очищен|
|`Failed`|Ошибка|Не удалось очистить или переиспользовать|

**Статусы PersistentVolumeClaim:**

|Статус|Описание|Что это значит|
|---|---|---|
|`Pending`|Ожидание|Ищет подходящий PV или создаёт новый|
|`Bound`|Привязан|Нашёл и привязался к PV|
|`Lost`|Потерян|PV к которому был привязан, удалён|

### 2.2. Access Modes (режимы доступа)

**Access Mode** — это режим доступа к хранилищу. Определяет сколько подов и как могут использовать том.  
**Три режима доступа:**

|Mode|Сокращение|Описание|Пример использования|
|---|---|---|---|
|ReadWriteOnce|`RWO`|Чтение-запись **одним** узлом|База данных, которая работает на одном поде|
|ReadOnlyMany|`ROX`|Только чтение **многими** узлами|Статические файлы (конфиги, сертификаты)|
|ReadWriteMany|`RWX`|Чтение-запись **многими** узлами|Общая директория для логов от нескольких подов|

**Важно:** Не все типы хранилищ поддерживают все режимы!  
**Таблица поддержки по типам:**

|Тип хранилища|RWO|ROX|RWX|
|---|---|---|---|
|AWS EBS|✅|❌|❌|
|GCE Persistent Disk|✅|✅|❌|
|Azure Disk|✅|❌|❌|
|NFS|✅|✅|✅|
|CephFS|✅|✅|✅|
|HostPath|✅|❌|❌|

### 2.3. Reclaim Policy (политика освобождения)

**Reclaim Policy** определяет что происходит с PersistentVolume после удаления связанного PVC.  
**Три политики:**

|Политика|Что происходит|Когда использовать|
|---|---|---|
|`Retain`|PV остаётся, данные сохраняются, статус Released|На проде - нужна ручная очистка|
|`Delete`|PV и данные удаляются автоматически|На dev/test - автоматическая очистка|
|`Recycle`|Данные удаляются (`rm -rf`), PV становится Available|не надо использовать)|

**Важно:** `Recycle` устарел и не поддерживается. Используй только `Retain` или `Delete`.  
**Что происходит при удалении PVC:**

javascriptcopy

```javascript
# 1. Удаляем PVCkubectl delete pvc prod-pvc # 2. PV переходит в статус Released (не Available!)kubectl get pv prod-pv# NAME      CAPACITY   STATUS     CLAIM# prod-pv   100Gi      Released   default/prod-pvc # 3. Нужно вручную очистить и переиспользовать# - Удалить старые данные если нужно# - Удалить метаданные о старом PVC# - Или создать новый PV
```

### 2.4. Volume Binding Modes

**Volume Binding Mode** определяет **когда** происходит связывание PVC с PV.  
**Два режима:**

|Режим|Когда связывается|Зачем нужен|
|---|---|---|
|`Immediate`|Сразу при создании PVC|По умолчанию - быстрое выделение|
|`WaitForFirstConsumer`|Когда под начинает использовать PVC|Для топологических зависимостей|

**Проблема с Immediate:**

javascriptcopy

```javascript
1. PVC создан → Kubernetes выделяет PV на узле A2. Под создаётся → Scheduler размещает под на узле B3. ❌ Под не может запуститься - PV на другом узле!
```

**Решение с WaitForFirstConsumer:**

javascriptcopy

```javascript
1. PVC создан → Ждёт2. Под создаётся → Scheduler размещает под на узле B3. Kubernetes создаёт/выделяет PV на узле B ✅4. Под запускается успешно
```

## 3. Типы Persistent Volumes

Kubernetes поддерживает множество типов хранилищ. Разберём основные.

### 3.1. HostPath (локальный путь на ноде)

**Что это:** Директория или файл на узле кластера.  
**Когда использовать:**

- 🟡 Т**ОЛЬКО для разработки и тестирования
    
- 🟡 Single-node кластеры (minikube, kind)
    

**Типы HostPath:**

|Тип|Описание|
|---|---|
|`DirectoryOrCreate`|Создать директорию если не существует|
|`Directory`|Директория должна существовать|
|`FileOrCreate`|Создать файл если не существует|
|`File`|Файл должен существовать|
|`Socket`|Unix socket должен существовать|
|`BlockDevice`|Блочное устройство|

**Проблемы HostPath:**

- ❌ Данные на конкретной ноде - если под переедет на другой узел, то данных не будет
    
- ❌ Нет репликации
    
- ❌ Нет бэкапирования
    
- ❌ Проблемы с безопасностью (доступ к файловой системе ноды)
    

### 3.2. NFS (Network File System)

**Что это:** Сетевое хранилище, доступное по сети.  
**Когда использовать:**

- ✅ Нужен ReadWriteMany (несколько подов)
    
- ✅ Общая директория для логов, конфигов
    
- ✅ Старые on-premise инфраструктуры
    

**Преимущества:**

- ✅ Поддержка RWX - много подов могут писать одновременно
    
- ✅ Независимость от узлов кластера
    
- ✅ Дёшево (можно развернуть свой NFS сервер)
    

**Недостатки:**

- Медленнее, чем локальные диски
    
- Если NFS сервер упал - упало сразу всё)
    

## 4. StorageClass - динамическое создание томов

### 4.1. Что такое StorageClass?

**StorageClass** — это способ описать классы хранилищ, доступных в кластере. Позволяет автоматически создавать PV по требованию (Dynamic Provisioning).  
**Зачем нужен:**

- ✅ Не нужно вручную создавать PV
    
- ✅ Автоматическое выделение хранилища по требованию
    
- ✅ Разные классы хранилищ (fast SSD, slow HDD, replicated)
    
- ✅ Управление параметрами хранилища (IOPS, тип диска)
    

**Как работает:**

javascriptcopy

```javascript
1. Создаём StorageClass (описываем тип хранилища)2. Потом кто-то создаёт PVC с указанием StorageClass3. Kubernetes автоматически создаёт PV через provisioner4. PVC связывается с созданным PV5. Под использует PVC
```

### 4.2. Структура StorageClass

javascriptcopy

```javascript
apiVersion: storage.k8s.io/v1kind: StorageClassmetadata:  name: fast-storage  # Имя классаprovisioner: kubernetes.io/aws-ebs  # Кто создаёт томаparameters:  # Параметры для provisioner  type: gp3  # Тип диска в AWS  iops: "3000"  throughput: "125"reclaimPolicy: Delete  # Что делать с PV после удаления PVCvolumeBindingMode: WaitForFirstConsumer  # Когда создавать томallowVolumeExpansion: true  # Можно ли увеличивать размерmountOptions:  # Опции монтирования  - debug
```

**Основные параметры:**

|Параметр|Описание|Значения|
|---|---|---|
|`provisioner`|Плагин который создаёт тома|`kubernetes.io/aws-ebs`, `kubernetes.io/gce-pd`, CSI драйвера|
|`parameters`|Специфичные параметры для provisioner|Зависит от типа хранилища|
|`reclaimPolicy`|Политика освобождения|`Delete` (по умолчанию), `Retain`|
|`volumeBindingMode`|Когда создавать том|`Immediate`, `WaitForFirstConsumer`|
|`allowVolumeExpansion`|Разрешить увеличение размера|`true`, `false`|

### 4.3. Использование StorageClass в PVC

**Указываем StorageClass в PVC:**

javascriptcopy

```javascript
apiVersion: v1kind: PersistentVolumeClaimmetadata:  name: fast-storage-claimspec:  accessModes:    - ReadWriteOnce  storageClassName: fast-ssd  # Используем StorageClass "fast-ssd"  resources:    requests:      storage: 50Gi
```

**Что происходит:**

javascriptcopy

```javascript
# 1. Создаём PVCkubectl apply -f pvc.yaml # 2. Kubernetes видит storageClassName: fast-ssd# 3. Находит StorageClass с именем fast-ssd# 4. Provisioner (ebs.csi.aws.com) создаёт диск в AWS# 5. Kubernetes автоматически создаёт PV# 6. PVC связывается с созданным PV
```

**Default StorageClass:**  
Можно пометить один StorageClass как default:

javascriptcopy

```javascript
apiVersion: storage.k8s.io/v1kind: StorageClassmetadata:  name: standard  annotations:    storageclass.kubernetes.io/is-default-class: "true"  # Класс по умолчаниюprovisioner: ebs.csi.aws.comparameters:  type: gp3
```

Теперь если не указать `storageClassName` в PVC, будет использован default:

javascriptcopy

```javascript
apiVersion: v1kind: PersistentVolumeClaimmetadata:  name: my-claimspec:  accessModes:    - ReadWriteOnce  # storageClassName не указан - использует default StorageClass  resources:    requests:      storage: 10Gi
```

## 5. Практические примеры

### База данных PostgreSQL с постоянным хранилищем

**Задача:** Развернуть PostgreSQL, данные должны сохраняться при перезапуске.  
**Решение:**

javascriptcopy

```javascript
# 1. PersistentVolumeClaim для PostgreSQLapiVersion: v1kind: PersistentVolumeClaimmetadata:  name: postgres-pvc  labels:    app: postgresspec:  accessModes:    - ReadWriteOnce  # База данных - одновременно один под  storageClassName: fast-ssd  # Используем быстрые диски  resources:    requests:      storage: 20Gi  # 20 ГБ для данных ---# 2. Deployment для PostgreSQLapiVersion: apps/v1kind: Deploymentmetadata:  name: postgres  labels:    app: postgresspec:  replicas: 1  # Один экземпляр (RWO)  selector:    matchLabels:      app: postgres  template:    metadata:      labels:        app: postgres    spec:      containers:      - name: postgres        image: postgres:15        env:        - name: POSTGRES_DB          value: myapp        - name: POSTGRES_USER          value: admin        - name: POSTGRES_PASSWORD          valueFrom:            secretKeyRef:              name: postgres-secret              key: password        ports:        - containerPort: 5432          name: postgres        volumeMounts:        - name: postgres-storage          mountPath: /var/lib/postgresql/data  # Директория данных PostgreSQL          subPath: postgres  # Подпапка в PV (важно для некоторых баз)      volumes:      - name: postgres-storage        persistentVolumeClaim:          claimName: postgres-pvc  # Наш PVC ---# 3. Service для доступа к PostgreSQLapiVersion: v1kind: Servicemetadata:  name: postgres  labels:    app: postgresspec:  type: ClusterIP  ports:  - port: 5432    targetPort: 5432    protocol: TCP  selector:    app: postgres
```

**Объяснение:**

- PVC запрашивает 20 ГБ на быстром SSD (`fast-ssd`)
    
- Kubernetes автоматически создаёт PV через StorageClass
    
- Deployment монтирует PVC в `/var/lib/postgresql/data`
    
- `subPath: postgres` создаёт подпапку (PostgreSQL требует пустую директорию)
    
- При перезапуске пода данные сохраняются
    

**Проверка:**

javascriptcopy

```javascript
# Создаём ресурсыkubectl apply -f postgres.yaml # Проверяем PVC - должен быть Boundkubectl get pvc postgres-pvc# NAME           STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS# postgres-pvc   Bound    pvc-abc123   20Gi       RWO            fast-ssd # Проверяем подkubectl get pods -l app=postgres# NAME                        READY   STATUS    RESTARTS# postgres-6f8d7c9b4d-xkj2p   1/1     Running   0 # Подключаемся и создаём данныеkubectl exec -it postgres-6f8d7c9b4d-xkj2p -- psql -U admin -d myapp# myapp=# CREATE TABLE test (id INT, name TEXT);# myapp=# INSERT INTO test VALUES (1, 'persistent data');# myapp=# SELECT * FROM test;#  id |      name# ----+-----------------#   1 | persistent data # Удаляем под (эмуляция перезапуска)kubectl delete pod postgres-6f8d7c9b4d-xkj2p # Deployment создаст новый подkubectl get pods -l app=postgres# NAME                        READY   STATUS    RESTARTS# postgres-6f8d7c9b4d-mnb5w   1/1     Running   0 # Подключаемся к новому поду - данные на месте!kubectl exec -it postgres-6f8d7c9b4d-mnb5w \  -- psql -U admin -d myapp -c "SELECT * FROM test;"#  id |      name# ----+-----------------#   1 | persistent data
```

### Пример 2: StatefulSet с постоянным хранилищем для каждой реплики

**Задача:** Развернуть Redis кластер, где каждая реплика имеет своё хранилище.  
**Решение:**

javascriptcopy

```javascript
# StatefulSet автоматически создаёт PVC для каждого подаapiVersion: apps/v1kind: StatefulSetmetadata:  name: redisspec:  serviceName: redis-headless  # Headless service для stable network ID  replicas: 3  # Три реплики Redis  selector:    matchLabels:      app: redis  template:    metadata:      labels:        app: redis    spec:      containers:      - name: redis        image: redis:7        command:        - redis-server        - --appendonly yes  # Включаем persistence        ports:        - containerPort: 6379          name: redis        volumeMounts:        - name: redis-data  # Имя соответствует volumeClaimTemplates          mountPath: /data  # Директория данных Redis  # volumeClaimTemplates создаёт PVC для каждого пода  volumeClaimTemplates:  - metadata:      name: redis-data  # Имя PVC    spec:      accessModes:        - ReadWriteOnce  # Каждый под - своё хранилище      storageClassName: fast-ssd      resources:        requests:          storage: 5Gi  # 5 ГБ на каждую реплику ---# Headless Service для StatefulSetapiVersion: v1kind: Servicemetadata:  name: redis-headlessspec:  clusterIP: None  # Headless  selector:    app: redis  ports:  - port: 6379    name: redis
```

**Объяснение:**

- `volumeClaimTemplates` автоматически создаёт PVC для каждого пода
    
- Имена PVC: `redis-data-redis-0`, `redis-data-redis-1`, `redis-data-redis-2`
    
- Каждый под имеет уникальное persistent хранилище
    
- При удалении пода, его PVC остаётся
    
- Новый под с тем же именем подключится к тому же PVC
    

### Пример 3: Общее хранилище для нескольких подов (ReadWriteMany)

**Задача:** Создать общую директорию для логов от нескольких микросервисов.  
**Решение:**

javascriptcopy

```javascript
# 1. StorageClass для NFS (поддерживает RWX)apiVersion: storage.k8s.io/v1kind: StorageClassmetadata:  name: nfs-storageprovisioner: example.com/nfs  # NFS provisioner (нужно установить)parameters:  server: nfs.company.local  path: /exports/k8svolumeBindingMode: Immediate ---# 2. PVC с ReadWriteManyapiVersion: v1kind: PersistentVolumeClaimmetadata:  name: shared-logs-pvcspec:  accessModes:    - ReadWriteMany  # Многие поды могут писать  storageClassName: nfs-storage  resources:    requests:      storage: 100Gi ---# 3. Deployment сервиса 1apiVersion: apps/v1kind: Deploymentmetadata:  name: service-aspec:  replicas: 3  selector:    matchLabels:      app: service-a  template:    metadata:      labels:        app: service-a    spec:      containers:      - name: app        image: myapp/service-a:1.0        volumeMounts:        - name: shared-logs          mountPath: /var/log/app          subPath: service-a  # Каждый сервис в своей подпапке      volumes:      - name: shared-logs        persistentVolumeClaim:          claimName: shared-logs-pvc ---# 4. Deployment сервиса 2apiVersion: apps/v1kind: Deploymentmetadata:  name: service-bspec:  replicas: 2  selector:    matchLabels:      app: service-b  template:    metadata:      labels:        app: service-b    spec:      containers:      - name: app        image: myapp/service-b:1.0        volumeMounts:        - name: shared-logs          mountPath: /var/log/app          subPath: service-b  # Своя подпапка      volumes:      - name: shared-logs        persistentVolumeClaim:          claimName: shared-logs-pvc  # Тот же PVC! ---# 5. Pod для сбора логовapiVersion: v1kind: Podmetadata:  name: log-aggregatorspec:  containers:  - name: aggregator    image: fluentd:latest    volumeMounts:    - name: shared-logs      mountPath: /logs  # Читает логи от всех сервисов      readOnly: true  # Только чтение  volumes:  - name: shared-logs    persistentVolumeClaim:      claimName: shared-logs-pvc
```

**Объяснение:**

- Один PVC с ReadWriteMany
    
- Все поды (service-a, service-b, log-aggregator) используют один PVC
    
- `subPath` создаёт отдельные подпапки для каждого сервиса
    
- Log aggregator читает логи от всех сервисов
    

**Результат:**

![Структура NFS-хранилища](https://offers.prostodevops.ru/diagrams/k8s/nfs-exports.png)Структура NFS-хранилища

## 6. Best Practices

**Для баз данных:**

- ✅ Облачные диски
    
- ✅ Быстрые SSD диски
    
- ✅ Access Mode: ReadWriteOnce
    
- ✅ StorageClass с высоким IOPS
    

**Для общих файлов (логи, конфиги):**

- ✅ NFS или CephFS
    
- ✅ Access Mode: ReadWriteMany
    
- ✅ Можно использовать медленные диски
    

**Для dev/test:**

- ✅ HostPath (только single-node кластеры)
    
- ✅ emptyDir для временных данных
    
- ✅ Reclaim Policy: Delete (автоматическая очистка)
    

**Для production:**

- ❌ НЕ использовать HostPath
    
- ✅ Reclaim Policy: Retain (ручная очистка)
    
- ✅ Включать шифрование (encryption)
    

## 7. Troubleshooting - типичные проблемы

### Проблема 1: PVC в статусе Pending

**Симптомы:**

javascriptcopy

```javascript
kubectl get pvc# NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS# my-pvc    Pending                                      fast-ssd kubectl describe pvc my-pvc# Events:#   Warning  ProvisioningFailed  waiting for a volume to be created
```

**Причины и решения:**  
**Причина 1: Нет подходящего PV и нет provisioner**

javascriptcopy

```javascript
# Проверь есть ли Available PVkubectl get pv# Нет Available PV → Создай PV вручную или настрой Dynamic Provisioning # Проверь StorageClasskubectl get sc fast-ssd# Если не существует - создай StorageClass # Проверь provisioner в StorageClasskubectl get sc fast-ssd -o yaml | grep provisioner# provisioner: kubernetes.io/no-provisioner  ← Ручное создание PV
```

**Решение:**

javascriptcopy

```javascript
# Если provisioner: kubernetes.io/no-provisioner# Создай PV вручную apiVersion: v1kind: PersistentVolumemetadata:  name: manual-pvspec:  capacity:    storage: 10Gi  accessModes:    - ReadWriteOnce  storageClassName: fast-ssd  hostPath:    path: /mnt/data
```

**Причина 2: Неправильный Access Mode**

javascriptcopy

```javascript
# PVC запрашивает ReadWriteMany# Но доступен только PV с ReadWriteOnce # Решение: измени accessModes в PVC или создай PV с нужным режимом
```

**Причина 3: Недостаточный размер**

javascriptcopy

```javascript
# PVC запрашивает 100Gi# Все доступные PV меньше (50Gi, 20Gi) # Решение: создай PV нужного размера или уменьши запрос в PVC
```

**Причина 4: Проблемы с CSI driver**

javascriptcopy

```javascript
# Проверь установлен ли CSI driverkubectl get pods -n kube-system | grep csi # Если нет - установи нужный driver (AWS EBS CSI, GCE PD CSI)
```

### Проблема 2: Pod не может смонтировать PVC

**Симптомы:**

javascriptcopy

```javascript
kubectl get pods# NAME           READY   STATUS              RESTARTS# my-pod         0/1     ContainerCreating   0 kubectl describe pod my-pod# Events:#   Warning  FailedMount  Unable to attach or mount volumes:#                            timeout expired waiting for volumes to attach
```

**Причины и решения:**  
**Причина 1: PV на другом узле (для локальных томов)**

javascriptcopy

```javascript
# Проверь где под и где PVkubectl get pod my-pod -o wide# NODE: worker-2 kubectl get pv pv-name -o yaml | grep nodeAffinity -A 10# nodeAffinity:#   required:#     nodeSelectorTerms:#     - matchExpressions:#       - key: kubernetes.io/hostname#         operator: In#         values:#         - worker-1  ← PV на worker-1, а под на worker-2! # Решение: используй volumeBindingMode: WaitForFirstConsumer# Или удали под - пересоздастся на правильном узле
```

**Причина 2: Том уже смонтирован в другой под (RWO)**

javascriptcopy

```javascript
# ReadWriteOnce том может быть на одном узле# Если два пода на разных узлах пытаются использовать - ошибка # Проверь другие подыkubectl get pods -A -o wide | grep my-pvc # Решение: удали другие поды или используй RWX том
```

**Причина 3: Недостаточно прав у CSI driver**

javascriptcopy

```javascript
# Проверь логи CSI driverkubectl logs -n kube-system -l app=ebs-csi-controller # Часто проблема с IAM правами в облаке# Дай CSI driver необходимые права (EC2, EBS API)
```

### Проблема 3: PV в статусе Released после удаления PVC

**Симптомы:**

javascriptcopy

```javascript
kubectl get pv# NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM# pv-1    50Gi       RWO            Retain           Released   default/old-pvc
```

**Причина:** Reclaim Policy = Retain, PVC удалён, но PV не может быть переиспользован.  
**Решение:**

javascriptcopy

```javascript
# Вариант 1: Очисти metadata и переиспользуй PVkubectl patch pv pv-1 -p '{"spec":{"claimRef": null}}' # Проверьkubectl get pv pv-1# NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM# pv-1    50Gi       RWO            Retain           Available # Теперь PV можно использовать снова # Вариант 2: Удали PV и создай новыйkubectl delete pv pv-1kubectl apply -f new-pv.yaml # Вариант 3: Если данные нужны# 1. Создай новый PV с теми же данными# 2. Измени path/volumeID на существующее хранилище# 3. Новый PVC привяжется к новому PV с старыми данными
```

### Проблема 4: PVC не удаляется (статус Terminating)

**Симптомы:**

javascriptcopy

```javascript
kubectl delete pvc my-pvc# persistentvolumeclaim "my-pvc" deleted # Но PVC всё ещё естьkubectl get pvc# NAME     STATUS        VOLUME# my-pvc   Terminating   pvc-abc123
```

**Причина:** PVC защищён finalizer или используется подом.  
**Решение:**

javascriptcopy

```javascript
# 1. Проверь используется ли PVCkubectl get pods -A -o yaml | grep my-pvc# Если да - удали поды сначала # 2. Проверь finalizerskubectl get pvc my-pvc -o yaml | grep finalizers -A 5# finalizers:# - kubernetes.io/pvc-protection  ← Защита от удаления пока используется # 3. Если под удалён, но PVC не удаляется - убери finalizerkubectl patch pvc my-pvc -p '{"metadata":{"finalizers":null}}' # PVC удалится немедленно
```

**Частые ошибки на собеседованиях:**

- Путают PV и PVC (кто что создаёт и для чего)
    
- Не понимают разницу между Access Modes (когда RWO, когда RWX)
    
- Не знают про Dynamic Provisioning и StorageClass
    
- Забывают про Reclaim Policy (данные удаляются по умолчанию!)
    

**Что запомнить:**

- PV = реальное хранилище, PVC = запрос на хранилище
    
- StorageClass = шаблон для автоматического создания PV
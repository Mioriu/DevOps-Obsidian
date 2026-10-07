# ConfigMap и Secrets в Kubernetes

## Где это может пригодиться:

- Хранение конфигурационных файлов приложений (nginx.conf, application.properties)
    
- Управление переменными окружения без пересборки образов
    
- Разделение конфигов для разных окружений (dev, test, prod)
    
- Хранение секретов - пароли БД, API ключи, токены
    
- Подключение сертификатов SSL/TLS к приложениям
    
- Изменение настроек приложений без рестарта подов
    
- Централизованное управление конфигурацией микросервисов
    

На собеседованиях могут спросить:

- В чём разница между ConfigMap и Secret?
    
- Как подключить ConfigMap к поду?
    
- Где физически хранятся Secret в кластере?
    
- Что происходит с подом при обновлении ConfigMap?
    
- Почему Secret небезопасен по умолчанию?
    

## 1. Что такое ConfigMap и Secret?

### ConfigMap

**ConfigMap** — это объект Kubernetes для хранения конфигурации в виде **пар ключ-значение.**  
**Зачем нужен:**

- ✅ Отделяешь конфигурацию от кода приложения
    
- ✅ Не нужно пересобирать образ при изменении настроек
    
- ✅ Используешь одинаковый образ для разных окружений
    

**Простой пример:**  
Вместо того чтобы хардкодить в коде:

javascriptcopy

```javascript
DATABASE_HOST = "mysql.prod.svc"LOG_LEVEL = "info"
```

Создаём ConfigMap:

javascriptcopy

```javascript
apiVersion: v1kind: ConfigMapmetadata:  name: app-configdata:  DATABASE_HOST: "mysql.prod.svc"  LOG_LEVEL: "info"
```

## И приложение читает эти переменные из окружения.

### Secret

**Secret** — это объект Kubernetes для хранения чувствительных данных (пароли, токены, ключи).  
**В чём разница с ConfigMap:**

|Параметр|ConfigMap|Secret|
|---|---|---|
|Назначение|Обычные настройки|Чувствительные данные|
|Хранение|Открытый текст|Base64 в etcd|
|Размер|До 1 МБ|До 1 МБ|
|Использование|Конфиги, файлы|Пароли, токены, сертификаты|

## **Важно:** Secret хранится в `Base64`, а **не шифруется** по умолчанию! `Base64` это кодирование, не шифрование.

## 2. Как создать ConfigMap

### Способ 1: Из литеральных значений

javascriptcopy

```javascript
# Создаём ConfigMap с переменнымиkubectl create configmap app-config \  --from-literal=DATABASE_HOST=mysql.prod.svc \  --from-literal=LOG_LEVEL=info
```

**Вывод:**

configmap/app-config created

Посмотреть содержимое:

kubectl get configmap app-config -o yaml

**Вывод:**

javascriptcopy

```javascript
apiVersion: v1kind: ConfigMapmetadata:  name: app-configdata:  DATABASE_HOST: mysql.prod.svc  LOG_LEVEL: info
```

### Способ 2: Из файла

Создаём конфиг nginx:

javascriptcopy

```javascript
cat > nginx.conf <<EOFserver {    listen 80;    location / {        proxy_pass http://backend:8080;    }}EOF
```

Создаём ConfigMap из файла:

kubectl create configmap nginx-config --from-file=nginx.conf

Проверяем:

kubectl describe configmap nginx-config

**Вывод:**

javascriptcopy

```javascript
Name:         nginx-configData====nginx.conf:----server {    listen 80;    location / {        proxy_pass http://backend:8080;    }}
```

### Способ 3: Из YAML манифеста

**configmap.yaml:**

javascriptcopy

```javascript
apiVersion: v1kind: ConfigMapmetadata:  name: app-config  namespace: productiondata:  # Простые переменные  DATABASE_HOST: "mysql.prod.svc"  DATABASE_PORT: "3306"  LOG_LEVEL: "info"   # Целый конфиг файл  application.properties: |    server.port=8080    spring.datasource.url=jdbc:mysql://mysql:3306/mydb    logging.level.root=INFO
```

Применяем:

kubectl apply -f configmap.yaml

## 3. Как создать Secret

### Способ 1: Из литеральных значений

javascriptcopy

```javascript
# Создаём Secret с паролем БДkubectl create secret generic db-secret \  --from-literal=username=admin \  --from-literal=password=SuperSecretPass123
```

**Вывод:**

secret/db-secret created

Посмотреть:

kubectl get secret db-secret -o yaml

**Вывод:**

javascriptcopy

```javascript
apiVersion: v1kind: Secretmetadata:  name: db-secrettype: Opaquedata:  username: YWRtaW4=  password: U3VwZXJTZWNyZXRQYXNzMTIz
```

**Важно:** Данные в Base64, расшифровать просто:

echo "U3VwZXJTZWNyZXRQYXNzMTIz" | base64 -d

**Вывод:**

SuperSecretPass123

### Способ 2: Из файлов (для сертификатов)

javascriptcopy

```javascript
# Создаём Secret с SSL сертификатомkubectl create secret tls nginx-tls \  --cert=server.crt \  --key=server.key
```

### Способ 3: Из YAML манифеста

**secret.yaml:**

javascriptcopy

```javascript
apiVersion: v1kind: Secretmetadata:  name: db-secret  namespace: productiontype: Opaquedata:  # Значения должны быть в Base64  username: YWRtaW4=  password: U3VwZXJTZWNyZXRQYXNzMTIz
```

**Как закодировать в Base64:**

javascriptcopy

```javascript
echo -n "admin" | base64| --- | --- | echo -n "SuperSecretPass123" | base64
```

Применяем:

kubectl apply -f secret.yaml

## 4. Как подключить к поду

### Вариант 1: Через переменные окружения

**deployment.yaml:**

javascriptcopy

```javascript
apiVersion: apps/v1kind: Deploymentmetadata:  name: myappspec:  replicas: 2  selector:    matchLabels:      app: myapp  template:    metadata:      labels:        app: myapp    spec:      containers:      - name: app        image: myapp:1.0         # Подключаем переменные из ConfigMap        env:        - name: DATABASE_HOST          valueFrom:            configMapKeyRef:              name: app-config              key: DATABASE_HOST         - name: LOG_LEVEL          valueFrom:            configMapKeyRef:              name: app-config              key: LOG_LEVEL         # Подключаем секреты        - name: DB_USERNAME          valueFrom:            secretKeyRef:              name: db-secret              key: username         - name: DB_PASSWORD          valueFrom:            secretKeyRef:              name: db-secret              key: password
```

**Объяснение:**

- `configMapKeyRef` — берём значение из ConfigMap
    
- `secretKeyRef` — берём значение из Secret
    
- Приложение видит обычные переменные окружения
    

### Вариант 2: Загрузить все ключи сразу

javascriptcopy

```javascript
apiVersion: apps/v1kind: Deploymentmetadata:  name: myappspec:  template:    spec:      containers:      - name: app        image: myapp:1.0         # Все переменные из ConfigMap        envFrom:        - configMapRef:            name: app-config         # Все переменные из Secret        - secretRef:            name: db-secret
```

## **Важно:** Все ключи из ConfigMap и Secret станут переменными окружения.

### Вариант 3: Через volume (монтирование файлов)

**deployment.yaml:**

javascriptcopy

```javascript
apiVersion: apps/v1kind: Deploymentmetadata:  name: nginxspec:  template:    spec:      containers:      - name: nginx        image: nginx:1.21        volumeMounts:        - name: config-volume          mountPath: /etc/nginx/nginx.conf          subPath: nginx.conf         - name: tls-volume          mountPath: /etc/nginx/ssl          readOnly: true       volumes:      # ConfigMap как файл      - name: config-volume        configMap:          name: nginx-config       # Secret как файлы      - name: tls-volume        secret:          secretName: nginx-tls
```

**Объяснение:**

- `volumeMounts` — куда монтировать
    
- `subPath` — взять конкретный файл (иначе монтируется вся директория)
    
- Secret монтируется как файлы с правами 0400
    

Проверяем внутри пода:

kubectl exec -it nginx-xxx -- ls -la /etc/nginx/ssl/

**Вывод:**

javascriptcopy

```javascript
total 0-r--------. 1 root root 1679 Oct 29 10:00 tls.crt-r--------. 1 root root 1704 Oct 29 10:00 tls.key
```

## 5. Типы Secret

Kubernetes поддерживает разные типы Secret:

|Тип|Назначение|Пример|
|---|---|---|
|`Opaque`|Любые данные|Пароли, токены|
|`kubernetes.io/tls`|TLS сертификаты|SSL для Ingress|
|`kubernetes.io/dockerconfigjson`|Авторизация в registry|Приватные образы|
|`kubernetes.io/basic-auth`|Basic авторизация|Логин/пароль|
|`kubernetes.io/ssh-auth`|SSH ключи|Git клонирование|

## 6. Обновление ConfigMap и Secret

### Что происходит при обновлении?

**Важно понимать:**  
**Переменные окружения (env, envFrom):**

- ❌ **НЕ обновляются** автоматически
    
- Нужен рестарт пода для применения изменений
    

**Volume монтирование:**

- ✅ Обновляются автоматически (через 30-60 секунд)
    
- Под **не перезапускается**
    
- Приложение должно само перечитывать файлы
    

### Как обновить ConfigMap

**Способ 1: kubectl edit**

kubectl edit configmap app-config

Изменяешь значения, сохраняешь. Если используется volume — подождать минуту, изменения применятся.  
**Способ 2: kubectl apply**  
Редактируешь `configmap.yaml`:

javascriptcopy

```javascript
apiVersion: v1kind: ConfigMapmetadata:  name: app-configdata:  DATABASE_HOST: "mysql.prod.svc"  LOG_LEVEL: "debug"  # Изменили с info на debug
```

Применяешь:

kubectl apply -f configmap.yaml

### Как перезапустить поды после изменения

**Способ 1: kubectl rollout restart**

javascriptcopy

```javascript
# Перезапуск Deploymentkubectl rollout restart deployment myapp
```

Kubernetes создаст новые поды с обновлёнными переменными.  
**Способ 2: Изменить манифест**  
Добавить аннотацию с датой:

javascriptcopy

```javascript
spec:  template:    metadata:      annotations:        configmap-updated: "2025-10-29"
```

И применить. Kubernetes увидит изменения и пересоздаст поды.  
**Способ 3: Использовать immutable ConfigMap**

javascriptcopy

```javascript
apiVersion: v1kind: ConfigMapmetadata:  name: app-config-v2  # Новое имя!immutable: truedata:  DATABASE_HOST: "mysql.prod.svc"  LOG_LEVEL: "debug"
```

- ✅ ConfigMap нельзя изменить после создания
    
- ✅ При изменении создаёшь новый ConfigMap с новым именем
    
- ✅ Обновляешь Deployment чтобы использовать новый ConfigMap
    

## 7. Практический пример

### Nginx с SSL и конфигом

**Задача:** Развернуть Nginx с кастомным конфигом и TLS сертификатом.  
**Решение:**  
Создаём конфиг:

javascriptcopy

```javascript
cat > nginx.conf <<EOFserver {    listen 443 ssl;    server_name myapp.example.com;     ssl_certificate /etc/nginx/ssl/tls.crt;    ssl_certificate_key /etc/nginx/ssl/tls.key;     location / {        proxy_pass http://backend:8080;        proxy_set_header Host \$host;    }}EOF
```

Создаём ConfigMap и Secret:

javascriptcopy

```javascript
# ConfigMap с конфигомkubectl create configmap nginx-config --from-file=nginx.conf # Secret с сертификатомkubectl create secret tls nginx-tls \  --cert=server.crt \  --key=server.key
```

**nginx-deployment.yaml:**

javascriptcopy

```javascript
apiVersion: apps/v1kind: Deploymentmetadata:  name: nginxspec:  replicas: 2  selector:    matchLabels:      app: nginx  template:    metadata:      labels:        app: nginx    spec:      containers:      - name: nginx        image: nginx:1.21        ports:        - containerPort: 443         volumeMounts:        # Монтируем конфиг        - name: config          mountPath: /etc/nginx/conf.d/default.conf          subPath: nginx.conf         # Монтируем сертификаты        - name: tls          mountPath: /etc/nginx/ssl          readOnly: true       volumes:      - name: config        configMap:          name: nginx-config       - name: tls        secret:          secretName: nginx-tls
```

**Объяснение:**

- Nginx читает конфиг из `/etc/nginx/conf.d/default.conf`
    
- SSL сертификаты в `/etc/nginx/ssl/` с правами 0400
    
- При обновлении конфига nginx перечитает его через ~60 секунд
    

## 8. Best Practices

### Безопасность Secret

- ✅ Включить шифрование etcd
    
- ✅ Ограничить доступ через RBAC - не всем нужны Secret
    
- ✅ Использовать внешние хранилища
    
- ✅ Регулярно ротировать секреты
    
- ❌ Не логировать содержимое Secret
    
- ❌ Не монтировать Secret если можно обойтись переменными
    

**Как включить шифрование etcd:**

javascriptcopy

```javascript
# /etc/kubernetes/manifests/kube-apiserver.yamlspec:  containers:  - command:    - kube-apiserver    - --encryption-provider-config=/etc/kubernetes/enc/config.yaml
```

### Организация конфигурации

- ✅ Один ConfigMap на приложение
    
- ✅ Разделять по окружениям: `app-config-dev`, `app-config-prod`
    
- ✅ Хранить конфиги в Git вместе с манифестами
    
- ✅ Использовать Helm или Kustomize для управления конфигами
    
- ❌ Не создавай огромные ConfigMap (больше 1 МБ не влезет)
    
- ❌ Не хранить в ConfigMap чувствительные данные)))))))))))))))
    

### Управление изменениями

- ✅ Используй `immutable: true` для ConfigMap в проде
    
- ✅ Версионировать ConfigMap и Secret
    
- ✅ Автоматизировать рестарт подов при изменении конфига
    
- ✅ Хранить историю изменений в Git
    
- ❌ Не редактировать ConfigMap напрямую в проде через `kubectl edit`
    

## 9. Troubleshooting

### Проблема 1: ConfigMap не обновляется в поде

**Симптомы:**  
Изменил ConfigMap, но приложение использует старые значения.  
**Причины и решения:**  
**Причина 1: Используются переменные окружения (env/envFrom)**  
Переменные окружения устанавливаются при старте пода и не обновляются.  
**Решение:**

javascriptcopy

```javascript
# Перезапустить Deploymentkubectl rollout restart deployment myapp
```

**Причина 2: Монтирование через volume, но приложение не перечитывает файл**  
ConfigMap обновился в файловой системе, но приложение не заметило.  
**Решение:**  
Приложение должно само следить за изменениями файла или использовать библиотеки вроде:

- Spring Cloud Config (автоматический reload)
    
- Viper для Go (watch файлов)
    
- Python watchdog
    

Или перезапустить:

kubectl rollout restart deployment myapp

### Проблема 2: Secret не подключается к поду

**Симптомы:**

Error: couldn't find key username in Secret default/db-secret

**Причины и решения:**  
**Причина 1: Опечатка в имени ключа**  
Проверяем что в Secret:

kubectl get secret db-secret -o yaml

Смотрим ключи в разделе `data`. Если там `db\_username`, а ты пишешь `username` — будет ошибка.  
**Решение:**  
Исправить имя ключа в манифесте:

javascriptcopy

```javascript
env:- name: DB_USERNAME  valueFrom:    secretKeyRef:      name: db-secret      key: db_username  # Правильное имя
```

**Причина 2: Secret в другом namespace**  
Secret находится в namespace `production`, а под в `default`.  
**Решение:**  
Создать Secret в том же namespace что и под:

javascriptcopy

```javascript
kubectl create secret generic db-secret \  --from-literal=username=admin \  --from-literal=password=pass123 \  -n default
```

### Проблема 3: Под не запускается - ошибка mount

**Симптомы:**

MountVolume.SetUp failed for volume "config-volume" : configmap "app-config" not found

**Причины и решения:**  
**Причина: ConfigMap не существует**  
Проверяем:

kubectl get configmap app-config

Если нет - создаём:

kubectl apply -f configmap.yaml

**Важно:** ConfigMap должен быть создан **до** применения Deployment.  
**Правильный порядок применения:**

javascriptcopy

```javascript
# Сначала конфигkubectl apply -f configmap.yamlkubectl apply -f secret.yaml # Потом приложениеkubectl apply -f deployment.yaml
```

### Проблема 4: Secret в Base64 не читается приложением

**Симптомы:**  
Приложение получает строку вроде `YWRtaW4=` вместо `admin`.  
**Причины и решения:**  
**Причина: Используется volume монтирование**  
При монтировании через volume Kubernetes **автоматически декодирует** Base64. Но иногда проблемы с настройками.  
**Проверка:**

kubectl exec -it myapp-xxx -- cat /etc/secrets/username

Если видишь `YWRtaW4=` — проблема.  
**Решение 1: Использовать переменные окружения**  
Kubernetes автоматически декодирует:

javascriptcopy

```javascript
env:- name: USERNAME  valueFrom:    secretKeyRef:      name: db-secret      key: username
```

**Решение 2: Проверить что Secret правильно создан**  
Если создавал через YAML, используй `stringData` вместо `data`:

javascriptcopy

```javascript
apiVersion: v1kind: Secretmetadata:  name: db-secrettype: OpaquestringData:  # Не нужно кодировать в Base64  username: admin  password: pass123
```

**Что запомнить:**

- ConfigMap для обычных настроек, Secret для чувствительных данных
    
- Secret в Base64, не зашифрован - нужно включать encryption at rest
    
- Переменные окружения (env) не обновляются без рестарта пода
    
- Volume монтирование обновляется автоматически через ~60 секунд
    
- Secret монтируется с правами 0400 (только чтение для владельца)
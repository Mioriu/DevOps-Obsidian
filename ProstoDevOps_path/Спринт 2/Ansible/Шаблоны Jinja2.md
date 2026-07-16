Пока что файлы на серверы мы просто копируем через модуль `copy` — как есть, один в один. А теперь представьте ситуацию: у вас есть nginx, и на каждом сервере конфиг чуть-чуть отличается — имя сервера своё, порт может быть другой, upstream разные. Можно, конечно, для каждого сервера держать отдельный файл конфига. Десять серверов — десять файлов. Поменяли что-то в одном — надо менять во всех.

Шаблоны решают эту проблему. Вы пишете один файл-шаблон, в нём вместо конкретных значений — переменные. Ansible берёт этот шаблон, подставляет значения для каждого конкретного сервера и кладёт готовый конфиг на место.

## Модуль template

Работает почти как `copy`, только перед тем как положить файл на сервер, прогоняет его через шаблонизатор Jinja2:

yamlcopy

```yaml
  tasks:    - name: Сгенерировать конфиг nginx      template:        src: templates/nginx.conf.j2        dest: /etc/nginx/nginx.conf        owner: root        group: root        mode: "0644"      notify: Перезагрузить nginx
```

Шаблоны принято класть в папку `templates/` и давать им расширение `.j2`. Это не обязательно, но так принято, и все сразу понимают, что это шаблон, а не готовый файл.

Параметры у `template` те же, что у `copy` — `dest`, `owner`, `group`, `mode`. Только вместо `src` с готовым файлом — `src` с шаблоном, который будет обработан.

## Синтаксис Jinja2

Jinja2 — это шаблонизатор, который Ansible использует под капотом. В нём три типа конструкций

### Переменные — двойные фигурные скобки

jinja2copy

```jinja2
server_name {{ domain }};
listen {{ http_port }};
root {{ app_dir }}/public;
```

Ansible подставит значения переменных. Если `domain = "app.company.com"` и `http_port = 80`, на выходе получится:

codecopy

```
server_name app.company.com;
listen 80;
root /opt/myapp/public;
```

### Условия — {% if %}

jinja2copy

```jinja2
{% if ssl_enabled %}listen 443 ssl;ssl_certificate {{ ssl_cert_path }};ssl_certificate_key {{ ssl_key_path }};{% else %}listen 80;{% endif %}
```

Если переменная `ssl_enabled` равна `true` — в конфиг попадёт блок с SSL. Если нет — просто `listen 80`. На выходе чистый конфиг без всяких Jinja2-конструкций.

Можно проверять существование переменной:

jinja2copy

```jinja2
{% if custom_error_page is defined %}error_page 500 502 503 504 {{ custom_error_page }};{% endif %}
```

Это полезно, когда переменная может быть не задана, и вы не хотите, чтобы шаблон падал

### Циклы — {% for %}

jinja2copy

```jinja2
{% for server in upstream_servers %}    server {{ server.host }}:{{ server.port }} weight={{ server.weight | default(1) }};{% endfor %}
```

Если `upstream_servers` — это список словарей, на выходе получится столько строк, сколько элементов в списке. Допустим, три сервера — три строки `server ...`.

Полезная штука — `loop.index` для нумерации:

jinja2copy

```jinja2
{% for user in app_users %}# Пользователь {{ loop.index }}: {{ user.name }}{% endfor %}
```

И `loop.last` для последнего элемента (удобно, когда не нужна запятая в конце):

jinja2copy

```jinja2
[{% for host in db_hosts %}  "{{ host }}"{% if not loop.last %},{% endif %} {% endfor %}]
```

### Комментарии — {# #}

jinja2copy

```jinja2
{# Этот комментарий не попадёт в итоговый файл #}server_name {{ domain }};
```

В отличие от обычных комментариев (типа `#` в nginx-конфиге), Jinja2-комментарии полностью вырезаются из результата

## Фильтрые

Фильтры в Jinja2 — это функции, которые трансформируют значение. Применяются через пайп `|`

### default — значение по умолчанию

jinja2copy

```jinja2
listen {{ http_port | default(80) }};| --- | --- | worker_processes {{ workers | default(ansible_processor_cores) }};log_level {{ log_level | default('warn') }};
```

Если переменная не задана и вы не поставили `default` — шаблон упадёт с ошибкой. А с `default` он просто возьмёт указанное значение. На практике ставьте `default` везде, где переменная может быть не задана

### join — склеить список в строку

jinja2copy

```jinja2
# переменная dns_servers: [8.8.8.8, 8.8.4.4]resolver {{ dns_servers | join(' ') }};# результат: resolver 8.8.8.8 8.8.4.4;
```

### to_json и to_yaml — сериализация

labels: {{ container_labels | to_json }}

Полезно, когда нужно вставить структуру данных в конфиг как JSON или YAML. Ansible возьмёт ваш словарь или список и красиво сериализует.

### regex_replace — замена по регулярке

{{ app_url | regex_replace('^https?://', '') }}

Например, из `https://app.company.com` получить `app.company.com`.

### int и round — математика

jinja2copy

```jinja2
worker_connections {{ (ansible_memtotal_mb / 4) | int }};| --- | --- | shared_buffers {{ (ansible_memtotal_mb * 0.25) | round | int }}MB
```

Факт `ansible_memtotal_mb` — число, но после деления может получиться float. Фильтр `int` превращает его обратно в целое число.

### upper, lower, capitalize — регистр

HOSTNAME={{ ansible_hostname | upper }}

## Практика: шаблон nginx.conf

Вот реальный пример шаблона nginx

jinja2copy

```jinja2
# {{ ansible_managed }}# Не редактируйте этот файл вручную — он генерируется Ansible user www-data;worker_processes {{ nginx_worker_processes | default(ansible_processor_cores) }};pid /run/nginx.pid; events {    worker_connections {{ nginx_worker_connections | default(1024) }};} http {    sendfile on;    tcp_nopush on;    keepalive_timeout 65;    types_hash_max_size 2048;     include /etc/nginx/mime.types;    default_type application/octet-stream;     access_log /var/log/nginx/access.log;    error_log /var/log/nginx/error.log; {% for site in nginx_sites %}    server {        listen {{ site.port | default(80) }};        server_name {{ site.domain }}; {% if site.ssl | default(false) %}        listen 443 ssl;        ssl_certificate /etc/ssl/certs/{{ site.domain }}.crt;        ssl_certificate_key /etc/ssl/private/{{ site.domain }}.key;{% endif %}         location / {            proxy_pass http://{{ site.upstream_host }}:{{ site.upstream_port }};            proxy_set_header Host $host;            proxy_set_header X-Real-IP $remote_addr;            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;{% if site.websocket | default(false) %}            proxy_http_version 1.1;            proxy_set_header Upgrade $http_upgrade;            proxy_set_header Connection "upgrade";{% endif %}        }    } {% endfor %}}
```

А переменные в `group_vars/webservers.yml`:

yamlcopy

```yaml
nginx_worker_processes: 4nginx_worker_connections: 2048nginx_sites:  - domain: app.company.com    port: 80    upstream_host: 127.0.0.1    upstream_port: 8080    ssl: false  - domain: api.company.com    port: 80    upstream_host: 127.0.0.1    upstream_port: 3000    ssl: true    websocket: true
```

Один шаблон, одни переменные — и на каждом сервере свой правильный конфиг. Добавить новый сайт — просто добавляете элемент в список `nginx_sites`
# Nginx

## Установка

Тут всё просто:

bashcopy

```bash
apt updateapt install nginx
```

После установки Nginx автоматически стартует и добавляется в автозапуск. Проверяем:

systemctl status nginx

Если видите `active (running)` — всё работает. Можно открыть в браузере IP сервера — увидите стандартную страницу «Welcome to nginx».

## Структура конфигов

Вот тут нужно разобраться, потому что конфигурация Nginx — это не один файл, а целая иерархия.![Структура /etc/nginx: nginx.conf, conf.d, sites-available, sites-enabled](https://offers.prostodevops.ru/diagrams/linux/nginx-config-tree.png)Структура /etc/nginx: nginx.conf, conf.d, sites-available, sites-enabled

### nginx.conf — главный файл

Это корень всей конфигурации. Обычно его не трогают, там глобальные настройки:

nginxcopy

```nginx
user www-data;worker_processes auto;pid /run/nginx.pid; events {    worker_connections 1024;} http {    include /etc/nginx/mime.types;        access_log /var/log/nginx/access.log;    error_log /var/log/nginx/error.log;     include /etc/nginx/conf.d/*.conf;    include /etc/nginx/sites-enabled/*;}
```

Обратите внимание на строки `include` в конце блока `http`. Именно они подтягивают все остальные конфиги. То есть, `nginx.conf` — это каркас, а реальные настройки сайтов лежат в подключаемых файлах.

- **worker_processes auto** — Nginx создаст столько воркер-процессов, сколько у вас ядер CPU. Каждый воркер обрабатывает соединения независимо.
- **worker_connections 1024** — сколько одновременных соединений может держать один воркер. Итого: воркеры × connections = максимум одновременных соединений.

### conf.d/ vs sites-available/ + sites-enabled/

Тут есть два подхода, и оба рабочие.

**conf.d/** — просто папка, в которую кладёте файлы с расширением `.conf`. Всё, что там лежит, автоматически подключается. Положили файл — он активен. Удалили — отключился. Простой и прямолинейный подход. Используется в CentOS/RHEL по умолчанию и многие предпочитают его на Ubuntu тоже.

**sites-available/ + sites-enabled/** — двухступенчатая схема. В `sites-available/` лежат конфиги всех сайтов, а в `sites-enabled/` — симлинки на те, которые активны. Это подход Debian/Ubuntu.

Чтобы активировать сайт:

ln -s /etc/nginx/sites-available/mysite.conf /etc/nginx/sites-enabled/

Чтобы деактивировать — просто удаляете симлинк из `sites-enabled/`. Конфиг в `sites-available/` остаётся на месте, можно включить обратно в любой момент.

На практике используйте то, что принято на вашем проекте. Оба подхода работают одинаково — Nginx подключает файлы через `include`, и ему без разницы, как вы организовали директории.

## Виртуальные хосты — блок server

Каждый сайт или приложение описывается блоком `server`. Один Nginx может обслуживать десятки сайтов — каждый в своём блоке, в своём файле.

Простой пример — статический сайт:

nginxcopy

```nginx
server {    listen 80;    server_name example.com www.example.com;     root /var/www/example.com;    index index.html;     location / {        try_files $uri $uri/ =404;    }}
```

Разберём построчно.

### listen

nginxcopy

```nginx
listen 80;            # слушать порт 80 (HTTP)listen 443 ssl;       # слушать порт 443 (HTTPS)listen [::]:80;       # слушать по IPv6
```

Порт, на котором Nginx принимает соединения для этого server-блока.

### server_name

nginxcopy

```nginx
server_name example.com www.example.com;server_name *.example.com;              # вайлдкардserver_name _;                           # дефолтный сервер (ловит всё)
```

По какому доменному имени Nginx определяет, какой `server`-блок использовать. Когда приходит запрос, Nginx смотрит заголовок `Host` и ищет подходящий `server_name`.

`server_name _` — это «ловушка для всего остального». Если ни один server-блок не подошёл по имени — запрос попадёт сюда. Удобно для дефолтной страницы или редиректа.

### root и index

nginxcopy

```nginx
root /var/www/example.com;index index.html index.htm;
```

- **root** — корневая директория сайта. Откуда Nginx берёт файлы для отдачи.
- **index** — какой файл отдавать, если запрошена директория. Запросили `/` — Nginx ищет `index.html` в `root`.

### location — маршрутизация запросов

`location` — это, по сути, правила: «если URL выглядит вот так — делай вот это». Внутри одного `server` может быть много `location`-блоков:

nginxcopy

```nginx
server {    listen 80;    server_name example.com;    root /var/www/example.com;     # Статические файлы — отдаём напрямую    location /static/ {        expires 30d;        add_header Cache-Control "public, immutable";    }     # Картинки — отдаём с кэшированием    location ~* \.(jpg|jpeg|png|gif|ico|svg)$ {| --- | --- | --- | --- | --- | --- |         expires 7d;    }     # API — проксируем на бэкенд    location /api/ {        proxy_pass http://127.0.0.1:8080;    }     # Всё остальное — отдаём index.html (для SPA)    location / {        try_files $uri $uri/ /index.html;    }}
```

Типы location:

- `location /api/` — префиксный матч. Все URL, начинающиеся с `/api/`.
- `location = /health` — точный матч. Только `/health`, ничего другого.
- `location ~* \.(jpg|png)$` — регулярное выражение, `~*` означает регистронезависимый матч. | --- | --- |

Порядок приоритетов: точный (`=`) → приоритетный префикс (`^~`) → регулярное выражение (`~`, `~*`) → обычный префикс. Если не помните приоритеты — ничего страшного, в повседневной работе обычно хватает обычных префиксов и `proxy_pass`.

### try_files

try_files $uri $uri/ =404;

Nginx пробует варианты по порядку: сначала ищет файл `$uri`, потом директорию `$uri/`, если ничего не нашёл — отдаёт 404. Для SPA-приложений (React, Vue) часто пишут `try_files $uri $uri/ /index.html` — если файл не найден, отдаём `index.html`, и фронтенд разруливает маршрутизацию сам.

## Reverse proxy — proxy_pass

Самый частый сценарий использования Nginx в devops. Приложение крутится на каком-то порту, Nginx проксирует на него запросы:

nginxcopy

```nginx
server {    listen 80;    server_name app.example.com;     location / {        proxy_pass http://127.0.0.1:8080;                proxy_set_header Host $host;        proxy_set_header X-Real-IP $remote_addr;        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;        proxy_set_header X-Forwarded-Proto $scheme;    }}
```

Что тут происходит: запрос приходит на Nginx (порт 80), Nginx отправляет его на `127.0.0.1:8080`, где крутится приложение. Ответ от приложения Nginx отдаёт клиенту.

Заголовки `proxy_set_header` — это важно. Без них приложение не будет знать реальный IP клиента (увидит `127.0.0.1` — IP самого Nginx), не будет знать оригинальный хост и протокол. Эти четыре заголовка — стандартный набор, который нужно ставить всегда при проксировании.

![Nginx reverse proxy: клиент → Nginx:80 → приложение:8080](https://offers.prostodevops.ru/diagrams/linux/nginx-reverse-proxy.png)Nginx reverse proxy: клиент → Nginx:80 → приложение:8080

Проксировать можно не только на localhost. Можно на другой сервер, на несколько серверов (балансировка), на Unix-сокет:

nginxcopy

```nginx
# На другой серверproxy_pass http://10.0.1.15:8080; # На Unix-сокет (часто для PHP-FPM, Gunicorn)proxy_pass http://unix:/var/run/app.sock; # Балансировка между серверамиupstream backend {    server 10.0.1.10:8080;    server 10.0.1.11:8080;    server 10.0.1.12:8080;} server {    location / {        proxy_pass http://backend;    }}
```

## reload vs restart

Вы поменяли конфиг Nginx. Нужно применить изменения. Тут два варианта, и разница принципиальная:

**restart** — убивает все процессы Nginx и запускает заново:

systemctl restart nginx

Все текущие соединения клиентов рвутся. Кто-то качал файл — обрыв. Кто-то ждал ответ от бэкенда — обрыв. На продакшене это неприемлемо.

**reload** — мастер-процесс перечитывает конфиг, плавно перезапускает воркеры:

systemctl reload nginx

Старые воркеры дообрабатывают текущие запросы и завершаются. Новые воркеры стартуют с новым конфигом. Клиенты ничего не замечают. Никакого даунтайма.

Под капотом `reload` отправляет мастер-процессу сигнал SIGHUP — мы это разбирали в теме сигналов.

Правило простое: **на продакшене — всегда reload, никогда restart**. `restart` только если Nginx полностью завис и не реагирует на reload, что бывает крайне редко.

## nginx -t — проверка синтаксиса

Перед тем как применять изменения, нужно проверить, что конфиг валидный. Потому что если в конфиге ошибка — `reload` тихо откатится и продолжит работать со старым конфигом (хорошо), а `restart` упадёт и не поднимется (плохо).

nginx -t

codecopy

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is oknginx: configuration file /etc/nginx/nginx.conf test is successful
```

Если ошибка:

codecopy

```
nginx: [emerg] unknown directive "proxy_passs" in /etc/nginx/conf.d/app.conf:8nginx: configuration file /etc/nginx/nginx.conf test failed
```

Сразу видно: файл, строка, в чём проблема. Исправляете, проверяете ещё раз, и только потом `reload`.

Правильная последовательность при любых изменениях конфига:

![Nginx безопасный reload: edit → nginx -t → systemctl reload](https://offers.prostodevops.ru/diagrams/linux/nginx-reload.png)Nginx безопасный reload: edit → nginx -t → systemctl reload

Вбейте себе это в привычку. `nginx -t` перед каждым `reload`. Каждый раз. Без исключений.

## Логи: access.log и error.log

Nginx пишет два типа логов. Оба лежат по умолчанию в `/var/log/nginx/`.

### access.log

Сюда пишется каждый запрос, который пришёл на Nginx:

93.184.216.34 - - [29/Jan/2025:14:22:15 +0000] "GET /api/users HTTP/1.1" 200 1532 "https://example.com/" "Mozilla/5.0..."

Что тут: IP клиента, время, метод и URL запроса, код ответа (200), размер ответа в байтах, Referer, User-Agent. По этому логу можно понять, кто и что запрашивает, какие коды ответов получает, откуда идёт трафик.

Формат лога можно настроить:

nginxcopy

```nginx
http {    log_format main '$remote_addr - $remote_user [$time_local] '                    '"$request" $status $body_bytes_sent '                    '"$http_referer" "$http_user_agent" '                    '$request_time $upstream_response_time';     access_log /var/log/nginx/access.log main;}
```

`$request_time` и `$upstream_response_time` — полезные переменные. Первая — сколько времени Nginx потратил на обработку запроса целиком. Вторая — сколько ждал ответа от бэкенда. Если `upstream_response_time` большой — проблема в приложении. Если `request_time` большой, а `upstream_response_time` маленький — проблема в сети или у клиента.

Лог можно настроить для каждого server-блока отдельно:

nginxcopy

```nginx
server {    access_log /var/log/nginx/app.access.log main;}
```

Или отключить, если он не нужен (например, для health-check эндпоинта, который дёргается каждую секунду и забивает лог):

nginxcopy

```nginx
location /health {    access_log off;    return 200 "ok";}
```

### error.log

Сюда пишутся ошибки — проблемы с конфигом, ошибки подключения к бэкенду, таймауты, битые сертификаты:

codecopy

```
2025/01/29 14:25:03 [error] 1234#1234: *5678 connect() failed (111: Connection refused) while connecting to upstream, client: 93.184.216.34, upstream: "http://127.0.0.1:8080/api/users"
```

Вот этот лог — первое место, куда вы смотрите, когда что-то не работает. `Connection refused` — бэкенд упал или не слушает порт. `502 Bad Gateway` в браузере? Смотрите `error.log` — там будет написано, почему Nginx не смог достучаться до бэкенда.

Уровни логирования:

error_log /var/log/nginx/error.log warn;

Уровни: `debug`, `info`, `notice`, `warn`, `error`, `crit`, `alert`, `emerg`. По умолчанию `error`. Для отладки можно временно поставить `debug`, но на продакшене — `warn` или `error`, иначе лог будет расти очень быстро.

### Ротация логов

Nginx-овские логи — один из главных пожирателей места на диске. Мы уже разбирали это в теме дисков. `access.log` на нагруженном сервере может расти на гигабайты в день. При установке через пакетный менеджер logrotate обычно настраивается автоматически, но проверить стоит:

cat /etc/logrotate.d/nginx

Если конфиг на месте — нормально. Если нет — настраивайте руками, мы разбирали logrotate в теме дисков.

## Полезные команды для работы с Nginx

bashcopy

```bash
nginx -t                          # проверить синтаксис конфигаnginx -T                          # проверить + вывести весь результирующий конфигsystemctl reload nginx            # применить изменения без даунтаймаsystemctl status nginx            # статус + последние строки лога # Быстро посмотреть, кто ходитtail -f /var/log/nginx/access.log # Посмотреть ошибкиtail -f /var/log/nginx/error.log # Топ IP-адресов по количеству запросовawk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20| --- | --- | --- | --- | --- | # Топ URL-ов с 500-ми ошибкамиawk '$9 == 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20| --- | --- | --- | --- | --- |
```

`nginx -T` (с большой T) — полезная штука. Выводит полный результирующий конфиг со всеми include-ами в один поток. Удобно, когда конфиг раскидан по десяти файлам и нужно увидеть целую картину.
Web server и reverse proxy — это прослойка между интернетом и приложением. Она берёт на себя SSL, статику, маршрутизацию, защиту от некоторых атак.

## 1. Web server vs Application server

**Web server** (nginx, Apache) — отдаёт статический контент (HTML, CSS, JS, картинки), проксирует запросы к приложениям, терминирует SSL.

**Application server** (gunicorn, uvicorn, puma, Node.js) — выполняет код приложения, генерирует динамический контент.

Типичная схема:![Схема в материале «Сети расширенные 2: Web servers и reverse proxy»: Клиент → Nginx (web server) → Gunicorn (app server) → Django (приложение)](https://offers.prostodevops.ru/diagrams/extras/seti/seti-rasshirennye-2-web-servers-i-reverse-proxy-3.png)Схема в материале «Сети расширенные 2: Web servers и reverse proxy»: Клиент → Nginx (web server) → Gunicorn (app server) → Django (приложение)

Nginx быстро отдаёт статику, держит тысячи соединений, а тяжёлые динамические запросы передаёт приложению.

## 2. Forward proxy vs Reverse proxy

**Forward proxy** — прокси на стороне клиента. Клиент знает, что использует прокси. Используется для:

- выхода в интернет из корпоративной сети,
- обхода блокировок,
- анонимизации.

![Схема в материале «Сети расширенные 2: Web servers и reverse proxy»: Клиент → Forward Proxy → Интернет → Сервер](https://offers.prostodevops.ru/diagrams/extras/seti/seti-rasshirennye-2-web-servers-i-reverse-proxy-2.png)Схема в материале «Сети расширенные 2: Web servers и reverse proxy»: Клиент → Forward Proxy → Интернет → Сервер

Клиент настраивает прокси явно. Сервер не знает реальный IP клиента.

**Reverse proxy** — прокси на стороне сервера. Клиент не знает, что за прокси несколько серверов. Используется для:

- балансировки,
- SSL termination,
- кэширования,
- защиты backend-ов.

![Схема в материале «Сети расширенные 2: Web servers и reverse proxy»: Клиент → Интернет → Reverse Proxy → Backend серверы](https://offers.prostodevops.ru/diagrams/extras/seti/seti-rasshirennye-2-web-servers-i-reverse-proxy-1.png)Схема в материале «Сети расширенные 2: Web servers и reverse proxy»: Клиент → Интернет → Reverse Proxy → Backend серверы

Клиент думает, что общается с одним сервером. Backend-ы не торчат в интернет напрямую.

## 3. Что делает reverse proxy

### SSL/TLS termination

Reverse proxy расшифровывает HTTPS и передаёт на backend обычный HTTP.

![Reverse proxy: клиент → Nginx → Backend](https://offers.prostodevops.ru/diagrams/network/reverse-proxy.png)Reverse proxy: клиент → Nginx → Backend

Плюсы:

- Сертификаты в одном месте, а не на каждом backend-е
- Backend-ы проще (не нужно настраивать SSL)
- Можно использовать внутренние самоподписанные сертификаты или вообще HTTP внутри

### Отдача статики

Nginx отдаёт файлы напрямую с диска, не дёргая приложение:

nginxcopy

```nginx
location /static/ {    root /var/www;    expires 30d;} location / {    proxy_pass http://app:8080;}
```

Запрос `/static/style.css` → nginx отдаёт файл. Запрос `/api/users` → nginx проксирует на приложение.

### Маршрутизация

Разные URL → разные backend-ы:

nginxcopy

```nginx
location /api/ {    proxy_pass http://api-service:8080;} location /auth/ {    proxy_pass http://auth-service:8081;} location / {    proxy_pass http://frontend:3000;}
```

Reverse proxy может кэшировать ответы backend-а

### Защиты

- Rate limiting — ограничение запросов с одного IP
- Буферизация — защита от медленных клиентов
- Скрытие внутренней структуры — клиент не знает, сколько backend-ов и где они

## 4. Основные инструменты

### Nginx

Самый популярный. Простой конфиг, высокая производительность.

### Apache

Старше nginx, модульная архитектура. Чаще встречается в legacy.

### Caddy

Современный, автоматически получает SSL-сертификаты через Let's Encrypt.

### Traefik

Динамическая конфигурация, интеграция с Docker и Kubernetes. Популярен в контейнерных средах.

Traefik автоматически обнаруживает контейнеры и настраивает роутинг.

### Сравнение

|Инструмент|Сложность|Автоматический SSL|Динамическая конфигурация|Когда использовать|
|---|---|---|---|---|
|Nginx|Низкая|Нет (нужен certbot)|Нет|Большинство случаев|
|Apache|Средняя|Нет|Нет|Legacy, .htaccess|
|Caddy|Низкая|Да|Частично|Простые проекты, автоSSL|
|Traefik|Средняя|Да|Да|Docker, Kubernetes|

## 5. Важные заголовки при проксировании

Когда nginx проксирует запрос, backend видит соединение от nginx, а не от клиента. Нужно передавать информацию о клиенте через заголовки:

nginxcopy

```nginx
proxy_set_header Host $host;proxy_set_header X-Real-IP $remote_addr;proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;proxy_set_header X-Forwarded-Proto $scheme;
```

- **Host** — оригинальный домен из запроса клиента
- **X-Real-IP** — IP клиента
- **X-Forwarded-For** — цепочка прокси (клиент, прокси1, прокси2)
- **X-Forwarded-Proto** — протокол (http или https)

Приложение должно доверять этим заголовкам только от своего reverse proxy, иначе клиент может их подделать.

## Что запомнить

1. **Web server** отдаёт статику и проксирует, **application server** выполняет код
    
2. **Reverse proxy** — между интернетом и backend-ами. Клиент не знает, что за ним
    
3. **SSL termination** — HTTPS до прокси, HTTP после. Сертификаты в одном месте
    
4. **Nginx** — стандарт для большинства случаев. **Caddy** — если нужен автоматический SSL. **Traefik** — для Docker/Kubernetes
    
5. **Заголовки X-Real-IP, X-Forwarded-For** — передают информацию о клиенте backend-у
    
6. **Статику отдавай через nginx**, а не через приложение — так гораздо быстрее
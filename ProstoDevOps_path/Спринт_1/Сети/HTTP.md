## Зачем это знать?

HTTP — это протокол, на котором работает почти весь веб. REST API, GraphQL, веб-сокеты (начинаются с HTTP), загрузка файлов, webhooks. Понимание HTTP — обязательный навык для любого, кто работает с веб-инфраструктурой.

## 1. Что такое HTTP

HTTP (HyperText Transfer Protocol) — протокол прикладного уровня для передачи данных. Работает по модели запрос-ответ: клиент отправляет запрос, сервер возвращает ответ.

codecopy

```
Клиент                           Сервер   |                                |   |  GET /index.html HTTP/1.1      |   |  Host: example.com             |   |------------------------------->|   |                                |   |  HTTP/1.1 200 OK               |   |  Content-Type: text/html       |   |  <html>...</html>              |   |<-------------------------------|
```

HTTP — текстовый протокол (до версии 2). Можно читать запросы и ответы глазами, отлаживать через telnet.

Работает поверх TCP (обычно порт 80 для HTTP, 443 для HTTPS). В HTTP/3 — поверх QUIC (UDP).

## 2. Структура HTTP-запроса

![Схема в материале «Фундамент 4: HTTP»: GET /api/users?page=1 HTTP/1.1      ← стартовая строка (метод, путь, версия)](https://offers.prostodevops.ru/diagrams/extras/seti/fundament-4-http-6.png)Схема в материале «Фундамент 4: HTTP»: GET /api/users?page=1 HTTP/1.1 ← стартовая строка (метод, путь, версия)

### Методы HTTP

|Метод|Назначение|Тело запроса|Идемпотентный|Безопасный|
|---|---|---|---|---|
|GET|Получить ресурс|Нет|Да|Да|
|POST|Создать ресурс / отправить данные|Да|Нет|Нет|
|PUT|Заменить ресурс целиком|Да|Да|Нет|
|PATCH|Частично обновить ресурс|Да|Нет|Нет|
|DELETE|Удалить ресурс|Редко|Да|Нет|
|HEAD|Получить только заголовки|Нет|Да|Да|
|OPTIONS|Узнать поддерживаемые методы|Нет|Да|Да|

**Идемпотентный** — повторный запрос даёт тот же результат. GET, PUT, DELETE — идемпотентны. POST — нет (каждый POST может создавать новый ресурс).

**Безопасный** — не меняет состояние сервера. GET, HEAD, OPTIONS — безопасны.

### URL и его части

![Анатомия URL: scheme · host · port · path · query · fragment](https://offers.prostodevops.ru/diagrams/network/url-anatomy.png)Анатомия URL: scheme · host · port · path · query · fragment

- **Схема** — протокол (http, https, ftp)
- **Хост** — доменное имя или IP
- **Порт** — если не стандартный (80/443)
- **Путь** — идентификатор ресурса
- **Query** — параметры запроса
- **Fragment** — якорь (не отправляется на сервер)

## 3. Структура HTTP-ответа

![Схема в материале «Фундамент 4: HTTP»: HTTP/1.1 200 OK                     ← статусная строка (версия, код, reason)](https://offers.prostodevops.ru/diagrams/extras/seti/fundament-4-http-5.png)Схема в материале «Фундамент 4: HTTP»: HTTP/1.1 200 OK ← статусная строка (версия, код, reason)

### Коды ответов

Трёхзначное число. Первая цифра — категория:

|Категория|Диапазон|Значение|
|---|---|---|
|1xx|100-199|Информационные|
|2xx|200-299|Успех|
|3xx|300-399|Перенаправление|
|4xx|400-499|Ошибка клиента|
|5xx|500-599|Ошибка сервера|

### Коды, которые нужно знать наизусть

**2xx — успех:**

- **200 OK** — всё хорошо, вот ответ
- **201 Created** — ресурс создан (обычно ответ на POST)
- **204 No Content** — успех, но тела нет (часто ответ на DELETE)

**3xx — перенаправления:**

- **301 Moved Permanently** — ресурс переехал навсегда, обновите закладки
- **302 Found** — временное перенаправление
- **304 Not Modified** — ресурс не изменился, используйте кэш

**4xx — ошибки клиента:**

- **400 Bad Request** — некорректный запрос (невалидный JSON, отсутствует обязательное поле)
- **401 Unauthorized** — требуется аутентификация (плохой или отсутствующий токен)
- **403 Forbidden** — аутентификация есть, но доступ запрещён
- **404 Not Found** — ресурс не найден
- **405 Method Not Allowed** — метод не поддерживается (POST на endpoint, который принимает только GET)
- **408 Request Timeout** — клиент слишком долго отправлял запрос
- **429 Too Many Requests** — rate limit, слишком много запросов

**5xx — ошибки сервера:**

- **500 Internal Server Error** — что-то сломалось на сервере (необработанное исключение)
- **502 Bad Gateway** — прокси/балансировщик не смог получить ответ от upstream
- **503 Service Unavailable** — сервис временно недоступен (перегрузка, maintenance)
- **504 Gateway Timeout** — upstream не ответил вовремя

### Разница между 401 и 403

**401** — "Я не знаю, кто ты". Отсутствует или невалидный токен/credentials. Клиент должен аутентифицироваться.

**403** — "Я знаю, кто ты, но тебе нельзя". Аутентификация прошла, но у пользователя нет прав на этот ресурс.

### Разница между 502 и 504

**502 Bad Gateway** — nginx (или другой прокси) получил невалидный ответ от бэкенда. Бэкенд ответил, но чем-то странным.

**504 Gateway Timeout** — nginx не дождался ответа от бэкенда. Бэкенд молчит или обрабатывает слишком долго.

## 4. Важные заголовки

### Заголовки запроса

|Заголовок|Назначение|Пример|
|---|---|---|
|Host|Хост (обязателен в HTTP/1.1)|`Host: api.example.com`|
|User-Agent|Информация о клиенте|`User-Agent: Mozilla/5.0...`|
|Accept|Какие форматы принимает клиент|`Accept: application/json`|
|Accept-Encoding|Какое сжатие поддерживает|`Accept-Encoding: gzip, deflate`|
|Authorization|Данные аутентификации|`Authorization: Bearer token123`|
|Content-Type|Тип тела запроса|`Content-Type: application/json`|
|Content-Length|Размер тела в байтах|`Content-Length: 1234`|
|Cookie|Куки|`Cookie: session=abc123`|
|If-None-Match|Условный запрос по ETag|`If-None-Match: "etag123"`|
|If-Modified-Since|Условный запрос по дате|`If-Modified-Since: Mon, 15 Jan...`|

### Заголовки ответа

|Заголовок|Назначение|Пример|
|---|---|---|
|Content-Type|Тип содержимого|`Content-Type: text/html; charset=utf-8`|
|Content-Length|Размер тела|`Content-Length: 5678`|
|Content-Encoding|Сжатие|`Content-Encoding: gzip`|
|Cache-Control|Правила кэширования|`Cache-Control: max-age=3600`|
|ETag|Идентификатор версии ресурса|`ETag: "abc123"`|
|Last-Modified|Дата последнего изменения|`Last-Modified: Mon, 15 Jan...`|
|Set-Cookie|Установить куку|`Set-Cookie: session=xyz; HttpOnly`|
|Location|URL для редиректа|`Location: https://example.com/new`|
|WWW-Authenticate|Способ аутентификации (при 401)|`WWW-Authenticate: Bearer`|

### Content-Type: распространённые значения

|MIME-тип|Описание|
|---|---|
|`text/html`|HTML-страница|
|`text/plain`|Простой текст|
|`text/css`|CSS|
|`application/json`|JSON|
|`application/xml`|XML|
|`application/javascript`|JavaScript|
|`application/octet-stream`|Бинарные данные|
|`multipart/form-data`|Форма с файлами|
|`application/x-www-form-urlencoded`|Обычная форма|
|`image/png`, `image/jpeg`|Изображения|

## 5. Кэширование

HTTP имеет встроенные механизмы кэширования. Правильно настроенный кэш снижает нагрузку на сервер и ускоряет загрузку.

### Cache-Control

Главный заголовок для управления кэшем:

Cache-Control: max-age=3600, public

Директивы:

- `max-age=N` — кэшировать N секунд
- `no-cache` — можно кэшировать, но перед использованием проверить на сервере
- `no-store` — не кэшировать вообще (персональные данные)
- `public` — можно кэшировать на промежуточных прокси
- `private` — кэшировать только на клиенте (не на CDN)
- `immutable` — ресурс никогда не изменится (для версионированных файлов)

## 6. Keep-Alive и соединения

### Проблема HTTP/1.0

В HTTP/1.0 каждый запрос — новое TCP-соединение:

![Схема в материале «Фундамент 4: HTTP»: Запрос 1: TCP handshake → HTTP запрос → ответ → закрыть соединение](https://offers.prostodevops.ru/diagrams/extras/seti/fundament-4-http-4.png)Схема в материале «Фундамент 4: HTTP»: Запрос 1: TCP handshake → HTTP запрос → ответ → закрыть соединение

Три TCP handshake — это лишние round-trip'ы и задержка.

### Keep-Alive в HTTP/1.1

В HTTP/1.1 соединение по умолчанию остаётся открытым:

![Схема в материале «Фундамент 4: HTTP»: TCP handshake](https://offers.prostodevops.ru/diagrams/extras/seti/fundament-4-http-3.png)Схема в материале «Фундамент 4: HTTP»: TCP handshake

Заголовки:

codecopy

```
Connection: keep-alive       # держать соединение (по умолчанию в HTTP/1.1)Connection: close            # закрыть после ответаKeep-Alive: timeout=5, max=100  # параметры
```

### Head-of-Line Blocking

Проблема HTTP/1.1: запросы в одном соединении обрабатываются последовательно. Если первый запрос медленный, остальные ждут.

Обходные пути:

- Браузеры открывают 6-8 соединений к одному хосту
- Domain sharding — раздавать ресурсы с разных поддоменов

HTTP/2 решает эту проблему мультиплексированием.

## 7. HTTP/1.1, HTTP/2, HTTP/3

### HTTP/1.1 (1997)

- Текстовый протокол
- Одно соединение — один запрос за раз (или keep-alive с очередью)
- Head-of-line blocking
- Заголовки передаются полностью каждый раз (много overhead)

### HTTP/2 (2015)

- Бинарный протокол (эффективнее парсить)
- Мультиплексирование — много запросов одновременно в одном соединении
- Сжатие заголовков (HPACK)
- Server Push — сервер может отправить ресурсы до запроса
- Приоритеты потоков

![Схема в материале «Фундамент 4: HTTP»: Одно TCP-соединение:](https://offers.prostodevops.ru/diagrams/extras/seti/fundament-4-http-1.png)Схема в материале «Фундамент 4: HTTP»: Одно TCP-соединение:

### HTTP/3 (2022)

- Работает поверх QUIC (UDP), а не TCP
- Нет TCP head-of-line blocking (потеря пакета не блокирует другие потоки)
- Встроенное шифрование (TLS 1.3)
- Быстрее установка соединения (0-RTT в лучшем случае)
- Лучше работает на нестабильных сетях (мобильные)

### Как проверить версию

bashcopy

```bash
# curl покажет версиюcurl -I --http2 https://example.com# HTTP/2 200 # Или с verbosecurl -v --http2 https://example.com 2>&1 | grep "< HTTP"| --- | --- |
```

В браузере: DevTools → Network → правый клик на заголовках → Protocol.

## 8. HTTPS

HTTPS = HTTP + TLS. Шифрует весь трафик между клиентом и сервером.

### Что даёт TLS

- **Конфиденциальность** — трафик зашифрован, провайдер не видит содержимое
- **Целостность** — данные нельзя изменить незаметно
- **Аутентификация** — сертификат подтверждает, что это настоящий сервер

### Как проверить сертификат

bashcopy

```bash
# opensslopenssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -dates -subject| --- | --- | # curl покажет проблемы с сертификатомcurl -v https://example.com 2>&1 | grep -A5 "Server certificate"| --- | --- |
```

### Частые проблемы с HTTPS

**Сертификат истёк:**

curl: (60) SSL certificate problem: certificate has expired

**Сертификат для другого домена:**

curl: (60) SSL certificate problem: hostname mismatch

**Самоподписанный сертификат:**

curl: (60) SSL certificate problem: self signed certificate

Для тестирования можно игнорировать: `curl -k https://...` (но никогда в проде).

## 9. Диагностика HTTP

### curl — главный инструмент

bashcopy

```bash
# Простой GETcurl https://example.com # Показать заголовки ответаcurl -I https://example.com # Показать заголовки запроса и ответаcurl -v https://example.com # Только код ответаcurl -o /dev/null -s -w "%{http_code}\n" https://example.com # POST с JSONcurl -X POST https://api.example.com/users \  -H "Content-Type: application/json" \  -d '{"name": "John", "email": "john@example.com"}' # С авторизациейcurl -H "Authorization: Bearer token123" https://api.example.com/me # Следовать редиректамcurl -L https://example.com # Показать таймингиcurl -w "@curl-format.txt" -o /dev/null -s https://example.com # Через проксиcurl -x http://proxy:8080 https://example.com # Игнорировать проблемы с сертификатом (только для тестов!)curl -k https://self-signed.example.com
```

## 10. Типичные проблемы и решения

### Проблема 1: 502 Bad Gateway

Симптомы:

- Nginx/балансировщик возвращает 502
- В логах nginx: `upstream prematurely closed connection`

Причины:

- Бэкенд упал → перезапустить, проверить логи
- Бэкенд отвечает не тем (невалидный HTTP) → проверить приложение
- Неправильный upstream в конфиге → проверить адрес/порт

### Проблема 2: 504 Gateway Timeout

Симптомы:

- Запрос висит, потом 504
- В логах nginx: `upstream timed out`

Решения:

- Увеличить таймауты в nginx (если долгие запросы ожидаемы)
- Оптимизировать бэкенд
- Вынести долгие операции в фон (очереди)

### Проблема 3: 429 Too Many Requests

Симптомы:

- Внезапные 429 от API
- Заголовок `Retry-After` в ответе

Решения:

- Уменьшить частоту запросов
- Реализовать exponential backoff
- Проверить заголовки `X-RateLimit-*` для понимания лимитов
- Кэшировать ответы

### Проблема 4: CORS ошибки

Симптомы:

- В консоли браузера: "blocked by CORS policy"
- API работает из curl, но не из браузера

CORS — механизм безопасности браузера. Браузер не даёт JavaScript делать запросы к другому домену без разрешения сервера.

### Проблема 5: Redirect loop

Симптомы:

- Браузер: "too many redirects"
- curl -L зацикливается

Причины:

- HTTP → HTTPS → HTTP → ... (конфликт конфигов)
- www → non-www → www → ...
- Неправильная логика в приложении

## Что запомнить

1. **HTTP — запрос-ответ.** Методы: GET (получить), POST (создать), PUT (заменить), DELETE (удалить)
    
2. **Коды ответов:** 2xx — успех, 3xx — редирект, 4xx — ошибка клиента, 5xx — ошибка сервера
    
3. **502 — бэкенд ответил странно, 504 — бэкенд не ответил вовремя**
    
4. **Cache-Control управляет кэшированием.** `max-age`, `no-cache`, `no-store`
    
5. **HTTP/2 — мультиплексирование** (много запросов в одном соединении). HTTP/3 — поверх QUIC
    
6. **curl -v — главный инструмент диагностики**
    

## Шпаргалка

Методы:
GET     — получить
POST    — создать
PUT     — заменить целиком
PATCH   — частично обновить
DELETE  — удалить
HEAD    — только заголовки
OPTIONS — что поддерживается

Коды ответов:
200 OK              — успех
201 Created         — создано
204 No Content      — успех без тела
301 Moved           — переехало навсегда
302 Found           — временный редирект
304 Not Modified    — используй кэш
400 Bad Request     — кривой запрос
401 Unauthorized    — нужна аутентификация
403 Forbidden       — нет прав
404 Not Found       — не найдено
429 Too Many        — rate limit
500 Internal Error  — сервер сломался
502 Bad Gateway     — бэкенд ответил криво
503 Unavailable     — сервис недоступен
504 Gateway Timeout — бэкенд не ответил

Кэширование:
Cache-Control: max-age=3600        # кэшировать час
Cache-Control: no-cache            # проверять перед использованием
Cache-Control: no-store            # не кэшировать
ETag + If-None-Match               # условный запрос
Last-Modified + If-Modified-Since  # условный запрос

curl:
curl -I URL                # заголовки ответа
curl -v URL                # verbose (всё)
curl -X POST -d 'data' URL # POST
curl -H "Header: value"    # добавить заголовок
curl -L URL                # следовать редиректам
curl -o /dev/null -s -w "%{http_code}" URL  # только код
curl -k URL                # игнорировать TLS ошибки

Заголовки:
Host              — обязателен в HTTP/1.1
Content-Type      — тип тела (application/json)
Authorization     — токен (Bearer xxx)
Accept            — что клиент принимает
Cache-Control     — кэширование
Location          — URL редиректа

Версии HTTP:
HTTP/1.1 — текстовый, keep-alive, один запрос за раз
HTTP/2   — бинарный, мультиплексирование, сжатие заголовков
HTTP/3   — поверх QUIC (UDP), ещё быстрее

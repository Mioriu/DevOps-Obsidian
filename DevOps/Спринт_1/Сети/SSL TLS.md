# SSL/TLS

## Зачем это знать?

TLS — это то, что делает HTTPS безопасным. Понимание TLS нужно для настройки, отладки проблем с сертификатами, обеспечения безопасности инфраструктуры.

## 1. SSL и TLS: история

**SSL (Secure Sockets Layer)** — оригинальный протокол от Netscape, 1990-е годы. Версии 1.0 (никогда не выпускалась), 2.0, 3.0. Все версии SSL сейчас считаются небезопасными.

**TLS (Transport Layer Security)** — наследник SSL, разрабатывается IETF. Версии:

- TLS 1.0 (1999) — по сути SSL 3.1, устарел
- TLS 1.1 (2006) — устарел
- TLS 1.2 (2008) — всё ещё широко используется
- TLS 1.3 (2018) — текущий стандарт, быстрее и безопаснее

Когда говорят "SSL-сертификат" — имеют в виду сертификат для TLS. Термин SSL прижился, хотя протокол SSL мёртв.

|Протокол|Статус|Поддержка|
|---|---|---|
|SSL 2.0|Небезопасен|Отключать везде|
|SSL 3.0|Небезопасен (POODLE)|Отключать везде|
|TLS 1.0|Устарел|Отключать|
|TLS 1.1|Устарел|Отключать|
|TLS 1.2|Актуален|Поддерживать|
|TLS 1.3|Рекомендуется|Поддерживать|

## 2. Что даёт TLS

Данные зашифрованы. Никто между клиентом и сервером не может прочитать содержимое — ни провайдер, ни Wi-Fi-точка, ни злоумышленник.

Данные нельзя изменить незаметно. Если кто-то попытается модифицировать пакет, это будет обнаружено.

Сертификат подтверждает, что сервер — тот, за кого себя выдаёт. Ты точно подключился к `google.com`, а не к фишинговому сайту.

## 3. Как работает TLS Handshake

При установке TLS-соединения клиент и сервер договариваются о параметрах шифрования и обмениваются ключами. Это называется handshake.

### TLS 1.2 Handshake (упрощённо)

diagramcopy

```
Клиент                                        Сервер
   |                                             |
   |  1. ClientHello                             |
   |  (версии TLS, список шифров, random)        |
   |-------------------------------------------->|
   |                                             |
   |  2. ServerHello                             |
   |  (выбранная версия, шифр, random)           |
   |  3. Certificate (сертификат сервера)        |
   |  4. ServerKeyExchange (если нужен)          |
   |  5. ServerHelloDone                         |
   |<--------------------------------------------|
   |                                             |
   |  Клиент проверяет сертификат                |
   |                                             |
   |  6. ClientKeyExchange                       |
   |  (pre-master secret, зашифрованный          |
   |   публичным ключом сервера)                 |
   |  7. ChangeCipherSpec                        |
   |  8. Finished (зашифровано)                  |
   |-------------------------------------------->|
   |                                             |
   |  9. ChangeCipherSpec                        |
   |  10. Finished (зашифровано)                 |
   |<--------------------------------------------|
   |                                             |
   |  Соединение установлено, данные шифруются   |

```

Это 2 round-trip (4 пакета туда-обратно) до начала передачи данных.

### TLS 1.3 Handshake

TLS 1.3 быстрее — 1 round-trip:

diagramcopy

```
Клиент                                        Сервер
   |                                             |
   |  1. ClientHello + KeyShare                  |
   |-------------------------------------------->|
   |                                             |
   |  2. ServerHello + KeyShare                  |
   |  3. EncryptedExtensions                     |
   |  4. Certificate                             |
   |  5. CertificateVerify                       |
   |  6. Finished                                |
   |<--------------------------------------------|
   |                                             |
   |  7. Finished                                |
   |-------------------------------------------->|
   |                                             |
   |  Соединение установлено                     |

```

TLS 1.3 также поддерживает **0-RTT** (zero round-trip) для повторных соединений — данные можно отправить сразу с первым пакетом. Но 0-RTT уязвим для replay-атак, поэтому используется с осторожностью.

### Что происходит при проверке сертификата

Клиент получает сертификат сервера и проверяет:

1. **Срок действия** — сертификат не истёк и уже действует
2. **Имя хоста** — домен в сертификате совпадает с запрашиваемым
3. **Цепочка доверия** — сертификат подписан доверенным CA
4. **Не отозван** — сертификат не в списке отозванных (CRL/OCSP)

Если любая проверка провалилась — браузер показывает предупреждение.

## 4. Сертификаты

### Что внутри сертификата

Сертификат X.509 содержит:

- **Subject** — кому выдан (домен, организация)
- **Issuer** — кто выдал (Certificate Authority)
- **Validity** — срок действия (Not Before, Not After)
- **Public Key** — публичный ключ сервера
- **Extensions** — расширения (SAN, Key Usage и др.)
- **Signature** — подпись CA

bashcopy

```bash
# Посмотреть содержимое сертификатаopenssl x509 -in cert.pem -text -noout # Или скачать с сервера и посмотретьecho | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -text -noout
```

### Subject и SAN

**Subject** — основной домен сертификата:

Subject: CN=example.com

**SAN (Subject Alternative Name)** — дополнительные домены. Современные сертификаты используют SAN, а не CN:

codecopy

```
X509v3 Subject Alternative Name:    DNS:example.com, DNS:www.example.com, DNS:api.example.com
```

Один сертификат может покрывать несколько доменов через SAN.

### Wildcard-сертификаты

Сертификат на `*.example.com` покрывает любой поддомен первого уровня:

- ✓ `www.example.com`
- ✓ `api.example.com`
- ✓ `anything.example.com`
- ✗ `sub.api.example.com` (два уровня)
- ✗ `example.com` (без поддомена)

Для покрытия и `example.com`, и `*.example.com` нужны оба в SAN.

### Типы сертификатов по валидации

**DV (Domain Validation)** — CA проверяет только владение доменом. Выдаётся автоматически за минуты. Let's Encrypt выдаёт DV.

**OV (Organization Validation)** — CA проверяет организацию. Занимает дни. В сертификате указана компания.

**EV (Extended Validation)** — расширенная проверка организации. Раньше браузеры показывали зелёную строку с названием компании, теперь — нет. Практический смысл под вопросом.

Для большинства сайтов DV достаточно. Шифрование одинаковое, разница только в том, что проверял CA.

### Цепочка сертификатов

Сертификаты образуют цепочку доверия:

![Схема в материале «Фундамент 7: SSL/TLS»: Root CA (в браузере/ОС)](https://offers.prostodevops.ru/diagrams/extras/seti/fundament-7-ssl-tls-1.png)Схема в материале «Фундамент 7: SSL/TLS»: Root CA (в браузере/ОС)

**Root CA** — корневой сертификат, предустановлен в браузерах и ОС. Их немного (~100-150).

**Intermediate CA** — промежуточный. Root CA не подписывает сертификаты напрямую из соображений безопасности.

Сервер должен отдавать свой сертификат + все intermediate. Root отдавать не нужно — он уже есть у клиента.

bashcopy

```bash
# Проверить цепочкуopenssl s_client -connect example.com:443 -servername example.com # Покажет:# Certificate chain#  0 s:CN = example.com#    i:C = US, O = Let's Encrypt, CN = R3#  1 s:C = US, O = Let's Encrypt, CN = R3#    i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
```

### Форматы файлов

|Расширение|Формат|Содержимое|
|---|---|---|
|`.pem`|Base64 (текст)|Сертификат или ключ|
|`.crt`, `.cer`|Обычно PEM|Сертификат|
|`.key`|Обычно PEM|Приватный ключ|
|`.der`|Бинарный|Сертификат|
|`.pfx`, `.p12`|Бинарный (PKCS#12)|Сертификат + ключ вместе|

PEM-формат выглядит так:

codecopy

```
-----BEGIN CERTIFICATE-----MIIDdzCCAl+gAwIBAgIEAgAAuTANBgkqhkiG9w0BAQUFADBaMQswCQYDVQQGEwJJ... base64 ...-----END CERTIFICATE-----
```

## 5. Certificate Authority (CA)

CA — организация, которой доверяют браузеры и которая выпускает сертификаты.

### Публичные CA

|CA|Особенности|
|---|---|
|Let's Encrypt|Бесплатный, автоматический, DV only|
|DigiCert|Крупнейший коммерческий CA|
|Sectigo (Comodo)|Популярный, разные уровни|
|GlobalSign|Корпоративный сегмент|
|GoDaddy|Популярен у небольших сайтов|

### Let's Encrypt

Бесплатные сертификаты с автоматическим выпуском и обновлением. Certbot — основной клиент.

bashcopy

```bash
# Установка certbotapt install certbot python3-certbot-nginx # Получить сертификат для nginxcertbot --nginx -d example.com -d www.example.com # Или standalone (certbot сам поднимет веб-сервер)certbot certonly --standalone -d example.com # Обновить все сертификатыcertbot renew # Автоматическое обновление (обычно уже настроено)# cron: 0 0 * * * certbot renew --quiet# или systemd timer: certbot.timer
```

Сертификаты Let's Encrypt действуют 90 дней. Рекомендуется обновлять каждые 60 дней.

### Приватный CA

В корпоративных сетях часто используют свой CA для внутренних сервисов. Это требует добавления корневого сертификата в trust store всех клиентов.

bashcopy

```bash
# Создать приватный CA (упрощённо)# Генерируем ключ CAopenssl genrsa -out ca.key 4096 # Создаём корневой сертификатopenssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt \    -subj "/C=RU/O=My Company/CN=My Internal CA" # Добавить CA в trust store (Ubuntu/Debian)cp ca.crt /usr/local/share/ca-certificates/update-ca-certificates
```

## 6. Настройка TLS в nginx

### Важные директивы

**ssl_protocols** — какие версии TLS поддерживать. Только 1.2 и 1.3.

**ssl_ciphers** — список шифров. Используй рекомендации Mozilla.

**ssl_prefer_server_ciphers** — в TLS 1.3 не нужен, для TLS 1.2 иногда полезен.

**HSTS (Strict-Transport-Security)** — браузер запомнит, что сайт только HTTPS, и не будет пытаться HTTP.

**OCSP Stapling** — сервер сам получает подтверждение, что сертификат не отозван, и отдаёт клиенту. Быстрее, чем клиенту проверять самому.

## 8. Типичные проблемы и решения

### Проблема 1: Certificate has expired

Симптомы:

- Браузер: "Сертификат истёк"
- curl: `SSL certificate problem: certificate has expired`

Решение:

- Обновить сертификат: `certbot renew`
- Проверить, работает ли автообновление
- Настроить мониторинг срока действия

### Проблема 2: Hostname mismatch

Симптомы:

- curl: `SSL certificate problem: hostname mismatch`
- Браузер: "Сертификат выдан для другого сайта"

Причины:

- Сертификат на `example.com`, а сервис на `api.example.com`
- Wildcard `*.example.com` не покрывает `example.com`
- Обращение по IP вместо домена

Решение: перевыпустить сертификат с правильными SAN.

### Проблема 3: Unable to verify certificate

Симптомы:

- curl: `SSL certificate problem: unable to get local issuer certificate`
- Приложение не может подключиться

Причины:

- Сервер не отдаёт intermediate сертификат → добавить в конфиг
- Приватный CA не в trust store → добавить CA
- Устаревший CA bundle на клиенте → обновить

### Проблема 4: SSL handshake failure

Симптомы:

- `SSL_ERROR_HANDSHAKE_FAILURE_ALERT`
- Соединение обрывается при установке TLS

Причины:

- Нет общих шифров между клиентом и сервером
- Клиент не поддерживает версию TLS сервера
- SNI не отправляется, а сервер требует

### Проблема 5: Mixed content

Симптомы:

- Браузер блокирует часть контента
- Консоль: "Mixed Content: The page was loaded over HTTPS, but requested an insecure resource"

Причина: страница на HTTPS загружает ресурсы по HTTP.

Решение:

- Исправить ссылки на HTTPS
- Использовать protocol-relative URLs (`//example.com/...`) или относительные пути
- Добавить CSP header: `Content-Security-Policy: upgrade-insecure-requests`

## 9. mTLS (Mutual TLS)

Обычно только сервер показывает сертификат. В mTLS клиент тоже предъявляет сертификат серверу.

Используется:

- API между сервисами (service mesh)
- Корпоративные VPN
- Банковские API

## Что запомнить

1. **SSL мёртв, используем TLS.** Только TLS 1.2 и 1.3
    
2. **Сертификат содержит публичный ключ и подписан CA.** Цепочка: твой сертификат → intermediate → root
    
3. **Let's Encrypt — бесплатные сертификаты.** certbot для автоматизации, обновлять каждые 60 дней
    
4. **Сервер должен отдавать fullchain** (свой сертификат + intermediate)
    
5. **openssl s_client — главный инструмент диагностики.**
    
6. Истёкший сертификат = даунтайм
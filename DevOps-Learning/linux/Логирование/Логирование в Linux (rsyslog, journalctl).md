# Часть 1: Зачем нужны логи и где они живут

Представьте, что ваш сервер — это большой офис, где одновременно работают сотни сотрудников (процессов). Кто-то отвечает на звонки (веб-сервер), кто-то работает с документами (база данных), кто-то следит за безопасностью (SSH). И каждый из них ведёт свой журнал работы — это и есть **логи**.

## Зачем вообще нужны логи?

Логи решают три главные задачи:

1. **Расследование проблем** — когда что-то сломалось, логи расскажут, что произошло
2. **Безопасность** — кто и когда пытался войти в систему
3. **Аудит и соответствие** — для многих компаний хранение логов — это требование закона

## Где Linux хранит логи?

В Linux есть стандартное место для всех логов — директория `/var/log/`. Это как архив в офисе, где хранятся все журналы.

```bash
ls -la /var/log/
```

**Примерный вывод:**

```diff
total 12345
drwxr-xr-x  14 root   root     4096 Jul 16 10:30 .
drwxr-xr-x  13 root   root     4096 Jun 20 15:22 ..
drwxr-xr-x   2 root   root     4096 Jul 16 06:25 apt/
-rw-r-----   1 syslog adm     65432 Jul 16 10:30 auth.log
-rw-r-----   1 syslog adm    234567 Jul 16 10:25 kern.log
-rw-r-----   1 syslog adm    123456 Jul 16 10:30 syslog
-rw-rw-r--   1 root   utmp    28416 Jul 16 10:15 wtmp
-rw-r-----   1 syslog adm     12345 Jul 16 10:20 messages
drwxr-xr-x   2 root   root     4096 Jul 15 00:00 nginx/
drwxr-xr-x   2 mysql  adm      4096 Jul 14 12:00 mysql/
```

### **Разберём, что означает каждый файл:**

### Главные системные логи

**/var/log/syslog** (в Ubuntu/Debian) или **/var/log/messages** (в RHEL/CentOS)

- **Что там:** Почти все системные сообщения
- **Когда смотреть:** Первое место для поиска любых проблем
- **Пример записи:**

```less
Jul 16 10:30:45 myserver systemd[1]: Started Session 123 of user john.
Jul 16 10:30:46 myserver kernel: [12345.678] USB device disconnected
```

**/var/log/auth.log** (Ubuntu/Debian) или **/var/log/secure** (RHEL/CentOS)

- **Что там:** Все попытки входа в систему, использование sudo
- **Когда смотреть:** Проверка безопасности, проблемы с доступом
- **Пример записи:**

```javascript
Jul 16 10:30:45 myserver sshd[12345]: Accepted password for john from 192.168.1.100 port 22 ssh2
Jul 16 10:31:00 myserver sudo: john : TTY=pts/0 ; PWD=/home/john ; USER=root ; COMMAND=/bin/ls
Jul 16 10:31:15 myserver sshd[12346]: Failed password for invalid user hacker from 10.0.0.1 port 22 ssh2
```

**/var/log/kern.log**

- **Что там:** Сообщения от ядра Linux (драйверы, оборудование)
- **Когда смотреть:** Проблемы с железом, драйверами, модулями ядра
- **Пример записи:**

```yaml
Jul 16 10:30:00 myserver kernel: [12345.123] ata1.00: failed command: READ DMA
Jul 16 10:30:01 myserver kernel: [12345.124] sd 0:0:0:0: [sda] tag#0 FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_SENSE
```

### Логи конкретных сервисов

**/var/log/nginx/access.log и error.log**

- **access.log** — кто и что запрашивал с веб-сервера
- **error.log** — ошибки веб-сервера

```lua
# access.log
192.168.1.100 - - [16/Jul/2025:10:30:45 +0000] "GET /index.html HTTP/1.1" 200 1234 "-" "Mozilla/5.0"
error.log
2025/07/16 10:30:50 [error] 12345#12345: *100 open() "/var/www/html/missing.jpg" failed (2: No such file or directory)
```

### Специальные логи

**/var/log/dmesg**

- **Что там:** Сообщения загрузки системы
- **Когда смотреть:** Проблемы при старте, обнаружение оборудования

**/var/log/cron**

- **Что там:** Запуски запланированных задач
- **Когда смотреть:** Проверка выполнения cron-задач

### Практический пример: расследуем проблему

Пользователь жалуется, что не может зайти на сервер. Вот как мы ищем проблему:

```bash
# Смотрим последние 20 строк лога аутентификации
tail -20 /var/log/auth.log
```

**Вывод:**

```less
Jul 16 10:30:45 myserver sshd[12345]: Failed password for john from 192.168.1.100 port 22 ssh2
Jul 16 10:30:50 myserver sshd[12345]: Failed password for john from 192.168.1.100 port 22 ssh2
Jul 16 10:30:55 myserver sshd[12345]: Failed password for john from 192.168.1.100 port 22 ssh2
Jul 16 10:31:00 myserver sshd[12345]: Connection closed by authenticating user john 192.168.1.100 port 22 [preauth]
```

**Что мы видим:** Пользователь **john** трижды ввёл неправильный пароль. **Проблема найдена.**

Если нужно следить за логом в реальном времени (например, пока пользователь пытается войти):

```iecst
tail -f /var/log/auth.log
```

Команда `tail -f` будет показывать новые строки по мере их появления.

# Часть 2: Два способа работы с логами

В современном Linux есть **два основных способа работы с логами**. Это как иметь обычный бумажный журнал и электронную базу данных.

## Способ 1: Классические текстовые логи (rsyslog)

**rsyslog** — это классическая система, которая пишет логи в обычные текстовые файлы.

### Как работает rsyslog?

Представьте почтовое отделение:

1. **Программы** отправляют "_письма_" (сообщения) в rsyslog
2. `rsyslog` смотрит на "_адрес_" (facility) и "_важность_" (priority)
3. **По правилам** раскладывает письма в нужные ящики (файлы)

### Что такое facility и priority?

**Facility (источник) — кто отправил сообщение:**

- `auth/authpriv` — система безопасности (логины, sudo)
- `cron` — планировщик задач
- `daemon` — системные службы
- `kern` — ядро Linux
- `mail` — почтовая система
- `syslog` — сам демон syslog
- `user` — пользовательские программы (по умолчанию)
- `local0-local7` — для ваших собственных нужд

**Priority (важность) — насколько критично сообщение:**

Представьте светофор от зелёного к красному:

- `debug` (7) 🟢 — детальная отладочная информация
- `info` (6) 🟢 — обычные информационные сообщения
- `notice` (5) 🟡 — важные, но нормальные события
- `warning` (4) 🟡 — предупреждения, стоит обратить внимание
- `err` (3) 🟠 — ошибки, что-то не работает
- `crit` (2) 🔴 — критические ошибки
- `alert` (1) 🔴 — требуется немедленное вмешательство
- `emerg` (0) 🔴 — система неработоспособна!

**Важно:** Число в скобках — это код severity. Когда в конфигурации видите `*.crit`, это означает "уровень 2 и **ВЫШЕ** по важности" (то есть crit, alert, emerg).

## Способ 2: Современная система journald

**systemd-journald** — это новая система, которая хранит логи в специальной базе данных.

#### Преимущества journald

- **Структурированные данные** — можно искать по любому полю
- **Автоматическая ротация** — сам следит, чтобы не забить диск
- **Метаданные** — к каждому сообщению прикреплена куча полезной информации
- **Централизация** — все логи в одном месте

### Как они работают вместе?

В большинстве современных систем:

1. `journald` собирает **ВСЕ** сообщения
2. `journald` **может передавать** их в rsyslog
3. `rsyslog` записывает в **традиционные файлы**

Получается лучшее из двух миров!

# Часть 3: Учимся читать логи с journalctl

**journalctl** — это швейцарский нож для работы с логами. Давайте научимся им пользоваться на практических примерах.

## Базовые команды

**Посмотреть все логи:**

```nginx
journalctl
```

**Что вы увидите:**

```sql
-- Logs begin at Mon 2025-07-01 00:00:00 UTC, end at Wed 2025-07-16 10:45:00 UTC. --
Jul 01 00:00:00 myserver systemd[1]: Started System Logging Service.
Jul 01 00:00:01 myserver kernel: Linux version 5.15.0-75-generic
...тысячи строк...
Jul 16 10:45:00 myserver sshd[12345]: Accepted password for john from 192.168.1.100
```

**Проблема-** слишком много информации, поэтому учимся **фильтровать**.

## Навигация в journalctl

**journalctl** использует пейджер (как `less`). Горячие клавиши:

- `пробел` — страница вниз
- `b` — страница вверх
- `g` — в начало
- `G` — в конец
- `/текст` — поиск
- `q` — выход

## Полезные опции просмотра

**Показать логи с конца (последние события):**

```nginx
journalctl -e
```

Откроется в конце файла — удобно для свежих событий.

**Следить за логами в реальном времени:**

```nginx
journalctl -f
```

**Вывод в реальном времени:**

```nginx
Jul 16 10:45:00 myserver systemd[1]: Started OpenSSH server daemon.
Jul 16 10:45:01 myserver sshd[12345]: Server listening on 0.0.0.0 port 22.
Jul 16 10:45:05 myserver sshd[12346]: Accepted password for john from 192.168.1.100
^C (нажмите Ctrl+C для выхода)
```

**Показать последние N строк:**

```nginx
journalctl -n 50  # последние 50 строк
```

## Фильтрация по времени

Одна из самых мощных возможностей `journalctl` — **фильтрация по времени.**

**Логи за сегодня:**

```css
journalctl --since today
```

**Логи за последний час:**

```nginx
journalctl --since "1 hour ago"
```

**Логи за конкретный период:**

```bash
journalctl --since "2025-07-16 09:00:00" --until "2025-07-16 10:00:00"
```

**Комбинированные примеры:**

```bash
# Что происходило вчера с 14:00 до 15:00?
journalctl --since "yesterday 14:00" --until "yesterday 15:00"
# Последние 2 дня
journalctl --since "2 days ago"
# С прошлого понедельника
journalctl --since "last Monday"
```

## Фильтрация по важности

**Показать только ошибки и критические сообщения:**

```css
journalctl -p err
```

**Пример вывода:**

```vbscript
Jul 16 10:30:00 myserver nginx[12345]: [error] 123#123: *10 open() "/var/www/html/missing.jpg" failed
Jul 16 10:31:00 myserver kernel: [12345.678] ata1.00: failed command: READ DMA
Jul 16 10:32:00 myserver systemd[1]: Failed to start MySQL Community Server.
```

**Все уровни приоритета для фильтрации:**

```nginx
journalctl -p emerg   # только аварийные (система падает)
journalctl -p alert   # требуют немедленного вмешательства
journalctl -p crit    # критические ошибки
journalctl -p err     # обычные ошибки
journalctl -p warning # предупреждения
journalctl -p notice  # важные уведомления
journalctl -p info    # информационные сообщения
journalctl -p debug   # отладочная информация (ОЧЕНЬ много)
```

## Фильтрация по сервису

**Логи конкретного сервиса:**

```nginx
journalctl -u nginx.service
```

**Пример вывода:**

```less
Jul 16 10:00:00 myserver systemd[1]: Starting A high performance web server...
Jul 16 10:00:01 myserver nginx[12345]: nginx: the configuration file syntax is ok
Jul 16 10:00:01 myserver systemd[1]: Started A high performance web server.
Jul 16 10:30:00 myserver nginx[12345]: 2025/07/16 10:30:00 [notice] 12345#12345: signal 1 (SIGHUP) received, reconfiguring
```

**Можно вывести и для нескольких сервисов одновременно:**

```nginx
journalctl -u nginx.service -u mysql.service -u ssh.service
```

## Форматы вывода

`journalctl` может выводить данные в разных форматах. Это полезно для обработки скриптами.

**Обычный вывод (по умолчанию):**

```less
journalctl -u nginx -n 1
Jul 16 10:45:00 myserver nginx[12345]: 2025/07/16 10:45:00 [notice] 12345#12345: start worker processes
```

**Только сообщения без метаданных:**

```nginx
journalctl -u nginx -n 1 -o cat
2025/07/16 10:45:00 [notice] 12345#12345: start worker processes
```

**Подробный вывод со всеми полями:**

```yaml
journalctl -u nginx -n 1 -o verbose
Wed 2025-07-16 10:45:00.123456 UTC [s=1234567890abcdef;i=1a2b;b=3c4d5e6f;m=7890;t=abcdef;x=123456]
    PRIORITY=6
    _UID=0
    _GID=0
    _SYSTEMD_UNIT=nginx.service
    MESSAGE=2025/07/16 10:45:00 [notice] 12345#12345: start worker processes
    _PID=12345
    _COMM=nginx
    ...ещё десятки полей...
```

**JSON для программной обработки:**

```bash
journalctl -u nginx -n 1 -o json
{"__CURSOR":"s=1234...","__REALTIME_TIMESTAMP":"1626432300123456","PRIORITY":"6","MESSAGE":"2025/07/16 10:45:00 [notice]..."}
```

**Когда какой формат использовать:**

- `short` (по умолчанию) — для обычного просмотра
- `cat` — когда нужен только текст сообщений
- `verbose` — для глубокой отладки
- `json` — для обработки скриптами

## Поиск в логах

**Простой поиск по тексту:**

```nginx
journalctl -g "error"
```

Флаг `-g` работает как `grep` — ищет строки, содержащие указанный текст.

**Поиск с регулярными выражениями:**

```bash
# Найти все IP-адреса, начинающиеся с 192.168
journalctl -g "192\.168\.[0-9]+\.[0-9]+"
# Найти ошибки с кодами 4xx или 5xx
journalctl -u nginx -g "HTTP/[12].[01]"
```

# Часть 4: Настраиваем rsyslog под себя

`rsyslog` работает по правилам. Правило выглядит так: "_если пришло сообщение от такого-то источника с такой-то важностью, то записать его туда-то_".

## Где живут правила?

### **Главный файл конфигурации:**

```bash
cat /etc/rsyslog.conf
```

### **Что вы там увидите:**

```php
# Глобальные настройки
$FileOwner syslog
$FileGroup adm
$FileCreateMode 0640
# Загрузка модулей
module(load="imuxsock") # локальный системный сокет
module(load="imklog")   # сообщения ядра
# Правила по умолчанию
.info;mail.none;authpriv.none;cron.none     /var/log/messages
authpriv.                                    /var/log/secure
mail.*                                        /var/log/maillog
cron.*                                        /var/log/cron
# Включение дополнительных конфигураций
include(file="/etc/rsyslog.d/*.conf" mode="optional")
```

**Дополнительные правила (рекомендуемый способ):**

```bash
ls /etc/rsyslog.d/
20-ufw.conf  21-cloudinit.conf  50-default.conf
```

## Как читать правила rsyslog?

Формат правила: `facility.priority action`

**Разберём реальное правило:**

```iecst
auth,authpriv.*         /var/log/auth.log
```

Читается так: "Все сообщения `*` от подсистем `**auth**` И `**authpriv**` записывать в файл `/var/log/auth.log`"

**Более сложный пример:**

```iecst
*.info;mail.none;authpriv.none;cron.none     /var/log/messages
```

Читается так:

- `*.info` — все источники, уровень **info и выше**
- `mail.none` — НО НЕ от mail
- `authpriv.none` — И НЕ от authpriv
- `cron.none` — И НЕ от cron
- Результат пишем в `/var/log/messages`

## Создаём собственное правило

### **Задача:** У вас есть веб-приложение myapp, и вы хотите:

1. Все его логи писать в отдельный файл
2. Критические ошибки дублировать в специальный файл

### **Шаг 1: Создаём файл правил**

```bash
sudo nano /etc/rsyslog.d/50-myapp.conf
```

### **Шаг 2: Пишем правила**

```bash
# Правило 1: Все сообщения от myapp в отдельный файл
:programname, isequal, "myapp" /var/log/myapp/app.log
# Правило 2: Критические ошибки (severity <= 3) в отдельный файл
if $programname == 'myapp' and $syslogseverity <= 3 then {
action(type="omfile" file="/var/log/myapp/critical.log")
}
# Важно: прекратить дальнейшую обработку
:programname, isequal, "myapp" stop
```

**Что означает каждая строка:**

- `:programname, isequal, "myapp"` — если имя программы равно "**myapp**"
- `$syslogseverity <= 3` — уровень важности **3 или меньше** (err, crit, alert, emerg)
- `stop` — не обрабатывать это сообщение другими правилами

### **Шаг 3: Создаём директорию для логов**

```bash
sudo mkdir -p /var/log/myapp
sudo chown syslog:adm /var/log/myapp
```

### **Шаг 4: Перезапускаем rsyslog**

```bash
sudo systemctl restart rsyslog
```

### **Шаг 5: Тестируем**

```bash
# Отправляем тестовое сообщение
logger -t myapp "Приложение запущено"
# Отправляем критическую ошибку
logger -t myapp -p user.crit "Критическая ошибка базы данных!"
# Проверяем
cat /var/log/myapp/app.log
cat /var/log/myapp/critical.log
```

## Продвинутые возможности rsyslog

### **Отправка логов на удалённый сервер:**

```graphql
# По UDP (один @)
*.* @192.168.1.100:514
# По TCP (два @@) - надёжнее
. @@192.168.1.100:514
```

### **Фильтрация по содержимому сообщения:**

```1c
# Если сообщение содержит "ERROR"
:msg, contains, "ERROR" /var/log/errors.log
# Регулярное выражение
:msg, regex, "failed|error|critical" /var/log/problems.log
```

### **Динамические имена файлов:**

```bash
# Разные файлы для каждого дня
$template DailyLog,"/var/log/myapp/%$YEAR%-%$MONTH%-%$DAY%.log"
*.* ?DailyLog
```

# Часть 5: Настройка хранения логов

Логи **могут занять весь диск**, если их **не контролировать**. `journald` умеет сам следить за размером.

## Проверяем текущий размер логов

```css
journalctl --disk-usage
```

**Пример вывода:**

```vbnet
Archived and active journals take up 248.0M on disk.
```

### Файл настроек journald

```bash
sudo nano /etc/systemd/journald.conf
```

**Основные параметры с объяснениями:**

```ini
[Journal]
# Где хранить логи
Storage=persistent        # persistent - на диске, volatile - в RAM, auto - автовыбор

# Ограничения по размеру для постоянного хранилища (диск)
SystemMaxUse=1G          # Максимум места под логи (можно: 500M, 2G, 10% и т.д.)
SystemKeepFree=500M      # Минимум свободного места на диске

# Ограничения для временного хранилища (RAM)
RuntimeMaxUse=100M       # Максимум в RAM (для volatile storage)
RuntimeKeepFree=50M      # Минимум свободной RAM

# Ограничения по времени
MaxRetentionSec=1month   # Удалять логи старше месяца
# Варианты: 1day, 1week, 2months, 1year, infinity

# Размеры файлов журнала
SystemMaxFileSize=100M   # Максимальный размер одного файла
# После достижения создаётся новый файл

# Другие полезные параметры
Compress=yes            # Сжимать старые файлы (экономия места)
RateLimitBurst=1000     # Максимум сообщений за...
RateLimitInterval=30s   # ...указанный интервал (защита от спама)
```

### Применение настроек

```bash
# Перезапускаем journald
sudo systemctl restart systemd-journald
```

## Ручная очистка логов

### **Удалить логи старше определённого времени:**

```bash
# Оставить только последние 2 недели
sudo journalctl --vacuum-time=2weeks
# Оставить только последние 7 дней
sudo journalctl --vacuum-time=7d
# Оставить только последние 24 часа
sudo journalctl --vacuum-time=24h
```

**Пример вывода:**

```swift
Deleted archived journal /var/log/journal/.../system@...-...-0000000000000001-0005a8b3c1e5f123.journal (8.0M).
Deleted archived journal /var/log/journal/.../user-1000@...-...-0000000000000002-0005a8b3c1e5f456.journal (16.0M).
Vacuuming done, freed 24.0M of archived journals on disk.
```

### **Ограничить размер логов:**

```bash
# Оставить только 500MB
sudo journalctl --vacuum-size=500M
# Оставить только 1GB
sudo journalctl --vacuum-size=1G
```

### **Удалить всё кроме последних N файлов:**

```bash
# Оставить только 3 последних файла журнала
sudo journalctl --vacuum-files=3
```

## Условные рекомендации по настройке

**Для обычного компьютера:**

```ini
SystemMaxUse=500M        # Полгигабайта достаточно
MaxRetentionSec=1week    # Неделя истории
```

**Для сервера разработки:**

```ini
SystemMaxUse=2G          # Побольше места для отладки
MaxRetentionSec=1month   # Месяц для расследования проблем
```

**Для production сервера:**

```ini
SystemMaxUse=10G         # Много места для истории
MaxRetentionSec=6months  # Полгода для аудита
Compress=yes             # Обязательно сжимать
```

## Мониторинг состояния journald

**Проверить статус службы:**

```lua
systemctl status systemd-journald
```

**Посмотреть статистику:**

```css
journalctl --header
```

Покажет техническую информацию о файлах журнала.

## Попробуйте самостоятельно:

1. **Найдите все ошибки SSH за последние 24 часа:**
    
    ```css
    journalctl -u ssh.service -p err --since "24 hours ago"
    ```
    
2. **Настройте отдельный лог для критических ошибок всей системы:**
    - Создайте файл /etc/rsyslog.d/01-critical.conf
    - Добавьте правило: `*.crit /var/log/critical.log`
    - Перезапустите rsyslog
3. **Проверьте и оптимизируйте размер логов:**
    - Узнайте текущий размер: `journalctl --disk-usage`
    - Если больше 1GB, очистите: `sudo journalctl --vacuum-size=500M`

# Часть 4: Создаём свою конфигурацию

Допустим, у вас есть своё приложение, которое пишет логи. Давайте настроим для него ротацию.

### Шаг 1: Создаём конфигурацию

```bash
sudo nano /etc/logrotate.d/myapp
```

### Шаг 2: Пишем правила

```1c
/var/log/myapp/*.log {
    # Базовые настройки
    weekly          # ротировать раз в неделю
    rotate 4        # хранить 4 недели
    compress        # сжимать
    
    # Управление размером
    size 100M       # ротировать также если файл больше 100MB
    
    # Безопасность
    missingok       # не ругаться если файла нет
    notifempty      # не трогать пустые файлы
    
    # Создание нового файла
    create 644 myapp myapp    # права и владелец
    
    # Дополнительные фичи
    dateext         # добавить дату к имени (myapp.log-20250716)
    
    # Скрипты
    postrotate
        # Здесь можно перезапустить приложение
        # или отправить ему сигнал
        systemctl reload myapp || true
    endscript
}
```

## Тестирование конфигурации

**Опции командной строки logrotate:**

- `-d, --debug` — режим отладки, показывает что будет сделано, но НЕ выполняет ротацию
- `-f, --force` — принудительная ротация, даже если не пришло время
- `-v, --verbose` — подробный вывод, показывает детали процесса
- `-s, --state файл` — использовать альтернативный файл состояния (по умолчанию /var/lib/logrotate/status)
- `-m, --mail команда` — команда для отправки почты (по умолчанию /bin/mail)
- `-l, --log файл` — записать вывод logrotate в файл
- `--version` — показать версию

**Примеры использования:**

```bash
# Проверка синтаксиса без выполнения (debug mode)
sudo logrotate -d /etc/logrotate.d/myapp

# Принудительная ротация с подробным выводом
sudo logrotate -fv /etc/logrotate.d/myapp

# Использование альтернативного файла состояния
sudo logrotate -s /tmp/logrotate.status /etc/logrotate.conf

# Отладка + verbose для максимальной информации
sudo logrotate -dv /etc/logrotate.d/nginx

# Проверить все конфигурации
sudo logrotate -d /etc/logrotate.conf
```

## Полезные советы.

### Совет 1: Для активно пишущих приложений

Если приложение пишет **ОЧЕНЬ много логов:**

```1c
{
    hourly          # ротация каждый час!
    size 1G         # или при достижении 1GB
    rotate 24       # хранить сутки
    compress
    delaycompress
}
```

### Совет 2: Для критически важных логов

Если логи нужны для аудита:

```bash
{
    monthly
    rotate 24       # хранить 2 года
    compress
# Дополнительный бэкап
postrotate
    cp /var/log/audit/*.gz /backup/logs/
endscript
}
```

### Совет 3: Отладка проблем

Если ротация **не работает**:

1. Проверьте права на файлы и директории
2. Запустите с флагом `-v` (verbose): `sudo logrotate -v /etc/logrotate.conf`
3. Проверьте, есть ли ошибки в `/var/log/syslog`

**Полезные параметры:**

- **copytruncate** — для приложений, которые не могут переоткрыть лог-файл
- **maxage 30** — удалять логи старше 30 дней
- **minsize 1M** — ротировать только если размер больше 1MB
- **sharedscripts** — выполнить postrotate только один раз для всех файлов

# Часть 5: Где logrotate хранит информацию

## Состояние ротации

**`Logrotate`** хранит информацию о последней ротации каждого файла в `/var/lib/logrotate/status`:

```bash
cat /var/lib/logrotate/status
```

**Пример содержимого:**

```lua
logrotate state -- version 2
"/var/log/nginx/access.log" 2025-7-15-3:0:0
"/var/log/nginx/error.log" 2025-7-15-3:0:0
"/var/log/mysql/error.log" 2025-7-14-3:0:0
"/var/log/syslog" 2025-7-15-6:25:1
"/var/log/auth.log" 2025-7-15-6:25:1
```

**Формат записей:** `"путь/к/файлу" YYYY-M-D-H:M:S`

**Лайфхак:** если нужно сбросить историю для конкретного файла (например, для теста), просто **удалите его строку** из этого файла.

**Полезные команды для работы с состоянием:**

```bash
# Посмотреть когда был ротирован конкретный файл
grep "nginx" /var/lib/logrotate/status

# Сбросить состояние для конкретного файла (удалить строку)
sudo sed -i '/nginx\/access.log/d' /var/lib/logrotate/status

# Создать резервную копию состояния
sudo cp /var/lib/logrotate/status /var/lib/logrotate/status.backup
```

## Краткая шпаргалка по logrotate

- `logrotate -d config` — проверка конфигурации
- `logrotate -f config` — принудительная ротация
- `daily/weekly/monthly` — частота ротации
- `rotate N` — количество старых файлов
- `compress` — сжатие старых логов
- `size 100M` — ротация по размеру

## Итог

**Logrotate** — это страж дискового пространства, который автоматически управляет жизненным циклом лог-файлов. Правильно настроенная ротация позволяет хранить историю событий нужное время, не засоряя систему. Понимание работы logrotate критически важно для поддержания здоровья системы.


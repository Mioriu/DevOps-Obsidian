Традиционная система прав в Linux работает хорошо, но иногда её недостаточно. Представьте, что веб-сервер взломали через уязвимость в коде. Злоумышленник получил доступ от имени пользователя www-data. С точки зрения обычных прав, он теперь может читать все файлы этого пользователя, запускать процессы, обращаться к сети.

SELinux и AppArmor добавляют дополнительный уровень защиты — даже взломанный процесс сможет делать только то, что ему явно разрешено. 

### Проблема традиционных прав доступа

В Linux используется дискреционная модель контроля доступа (DAC — Discretionary Access Control). Это означает, что владелец файла сам решает, кто может его читать или изменять.

Рассмотрим типичный сценарий атаки. У вас есть веб-сервер `Apache`, который работает от пользователя `www-data`. Если в веб-приложении есть уязвимость, позволяющая выполнить произвольный код, злоумышленник получает все права пользователя `www-data`:

```haskell
Обычная система прав (DAC):
┌──────────────────────────────────────┐
│   Взломанный процесс Apache          │
│   Пользователь: www-data             │
│                                      │
│   Может:                             │
│   ✓ Читать все файлы www-data        │
│   ✓ Записывать в /tmp, /var/www      │
│   ✓ Открывать сетевые соединения     │
│   ✓ Запускать другие программы       │
│   ✓ Читать /etc/passwd               │
│   ✓ Обращаться к другим сервисам     │
└──────────────────────────────────────┘
```

Даже если вы настроили права файлов идеально, взломанный процесс всё равно может натворить много бед в рамках прав своего пользователя.

Мандатный контроль доступа (`MAC — Mandatory Access Control`) решает эту проблему, добавляя политики, которые **не может изменить даже владелец файла.**

### MAC vs DAC: принципиальная разница

Ключевое отличие `MAC` от `DAC` в том, кто контролирует доступ:

```bash
DAC (традиционная модель):
Пользователь → "Я владелец файла, я решаю кто может его читать"
              → chmod 777 myfile  (дал всем полный доступ)

MAC (SELinux/AppArmor):
Система → "Неважно, что ты владелец. Политика говорит, что Apache может читать только файлы с меткой httpd_content_t"
        → Даже chmod 777 не поможет, если метка неправильная
```

`MAC` работает поверх `DAC`. Сначала проверяются обычные права доступа, и только если они разрешают операцию, проверяется MAC-политика. Если **хотя бы одна** проверка не пройдена, **доступ запрещается**.

### SELinux

SELinux (Security-Enhanced Linux) был разработан АНБ США и является одной из самых мощных систем безопасности в Linux. Red Hat, CentOS и Fedora используют SELinux по умолчанию.  
Система работает на основе контекстов безопасности — специальных меток, которые присваиваются всем объектам в системе.

Каждый файл, процесс, порт и даже пользователь имеет контекст SELinux, состоящий из четырёх частей:

```1c
user:role:type:level
  │    │    │    │
  │    │    │    └── Уровень безопасности (s0, s0-s0:c0.c1023)
  │    │    └──────── Тип (самое важное - определяет доступ)
  │    └────────────── Роль (для пользователей и процессов)
  └──────────────────── Пользователь SELinux (не путать с Linux-пользователем)
```

Посмотрим на контексты в реальной системе:

```shell
# Контекст файлов
$ ls -Z /var/www/html/
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 about.html

# Контекст процессов
$ ps auxZ | grep httpd
system_u:system_r:httpd_t:s0   apache  2341  0.0  0.5  224936  5132 ?  S  10:20  0:00 /usr/sbin/httpd

# Контекст портов
$ sudo semanage port -l | grep http
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```

### Режимы работы SELinux

`SELinux` может работать в трёх режимах, что позволяет постепенно внедрять его в систему:

```bash
# Проверить текущий режим
$ getenforce
Enforcing

# Или более подробно
$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

**Три режима SELinux:**

- **Enforcing** — политики применяются, нарушения блокируются и логируются
- **Permissive** — политики не применяются, но нарушения логируются (режим отладки)
- **Disabled** — SELinux полностью отключен

**Изменение режима:**

```shell
# Временно переключить в Permissive (до перезагрузки)
$ sudo setenforce 0

# Обратно в Enforcing
$ sudo setenforce 1

# Постоянное изменение
$ sudo vi /etc/selinux/config
SELINUX=enforcing  # или permissive, или disabled

# После изменения с disabled на enforcing нужна перезагрузка
# и переразметка файловой системы
```

### Практическая работа с SELinux

Рассмотрим **типичную проблему:** вы изменили порт `SSH` с `22` на `2222`, но SELinux **блокирует** подключение.

```bash
# Изменили порт в /etc/ssh/sshd_config
Port 2222

# Перезапустили SSH
$ sudo systemctl restart sshd
Job for sshd.service failed. See "systemctl status sshd.service"

# Смотрим логи SELinux
$ sudo ausearch -m avc -ts recent
type=AVC msg=audit(1634567890.123:456): avc:  denied  { name_bind } for  
pid=1234 comm="sshd" src=2222 scontext=system_u:system_r:sshd_t:s0 
tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket

# SELinux говорит: sshd_t не может использовать порт типа unreserved_port_t
```

Решение — добавить порт `2222` в список разрешённых для `SSH`:

```bash
# Посмотреть, какие порты разрешены для SSH
$ sudo semanage port -l | grep ssh
ssh_port_t                     tcp      22

# Добавить новый порт
$ sudo semanage port -a -t ssh_port_t -p tcp 2222

# Проверить
$ sudo semanage port -l | grep ssh
ssh_port_t                     tcp      22, 2222

# Теперь SSH запустится успешно
$ sudo systemctl restart sshd
```

### Отладка проблем SELinux

SELinux часто обвиняют во всех проблемах, но есть инструменты для точной диагностики:

```bash
# Установить инструменты отладки
$ sudo yum install setroubleshoot-server

# После установки, SELinux будет давать подсказки в логах
$ sudo journalctl -xe
SELinux is preventing httpd from getattr access on the file /var/www/html/index.html.
***** Plugin restorecon suggests *****
If you want to fix the label, run:
# restorecon -v /var/www/html/index.html

# Использовать audit2why для анализа
$ sudo ausearch -m avc -ts recent | audit2why
Was caused by: Missing type enforcement (TE) allow rule.
You can use audit2allow to generate a loadable module to allow this access.

# Создать разрешающее правило (осторожно!)
$ sudo ausearch -m avc -ts recent | audit2allow -M mymodule
$ sudo semodule -i mymodule.pp
```

## AppArmor

AppArmor — это альтернатива SELinux, используемая по умолчанию в Ubuntu и SUSE. В отличие от SELinux с его контекстами, AppArmor работает с путями файлов, что делает его проще в понимании и настройке.

Основная идея AppArmor — создать профиль для каждого приложения, где явно указано, к каким файлам и ресурсам оно может обращаться:

```bash
# Проверить статус AppArmor
$ sudo aa-status
apparmor module is loaded.
34 profiles are loaded.
34 profiles are in enforce mode.
   /usr/bin/evince
   /usr/bin/firefox
   /usr/sbin/mysqld
   /usr/sbin/nginx
0 profiles are in complain mode.
3 processes have profiles defined.
```

### Режимы профилей AppArmor

Каждый профиль может находиться в одном из двух режимов:

- **Enforce** — правила применяются, нарушения блокируются
- **Complain** — нарушения только логируются, но не блокируются

```shell
# Переключить профиль в режим complain (для отладки)
$ sudo aa-complain /usr/sbin/nginx

# Вернуть в режим enforce
$ sudo aa-enforce /usr/sbin/nginx

# Отключить профиль
$ sudo aa-disable /usr/sbin/nginx
```

## Сравнение SELinux и AppArmor

Обе системы решают одну задачу, но разными способами. Вот ключевые отличия:

```scss
┌──────────────────┬────────────────────────┬────────────────────────┐
│                  │       SELinux          │       AppArmor         │
├──────────────────┼────────────────────────┼────────────────────────┤
│ Модель           │ Метки (контексты)      │ Пути файлов            │
│ Сложность        │ Высокая                │ Средняя                │
│ Гибкость         │ Очень высокая          │ Хорошая                │
│ Производительность│ Небольшие накладные   │ Минимальные накладные  │
│ По умолчанию в   │ RHEL, CentOS, Fedora   │ Ubuntu, SUSE           │
│ Переносимость    │ Требует переразметки   │ Работает сразу         │
└──────────────────┴────────────────────────┴────────────────────────┘
```

SELinux предоставляет более тонкий контроль и лучше подходит для критически важных серверов, где безопасность приоритетнее удобства. Каждый объект в системе имеет контекст, и политики определяют взаимодействие между контекстами. Это мощно, но требует глубокого понимания.

AppArmor проще в освоении и администрировании, так как работает с привычными путями файлов. Профили легче читать и модифицировать.

## Когда что выбирать?

Выбор между SELinux, AppArmor или отказом от MAC зависит от требований:

**SELinux, если:**

- Red Hat-based дистрибутивы
- Нужен максимальный уровень безопасности
- Управление критической инфраструктурой

**AppArmor, если:**

- Ubuntu или SUSE
- Нужен баланс безопасности и простоты

**Можно обойтись без MAC, если:**

- Это изолированная тестовая среда
- Накладные расходы критичны
- Нет ресурсов на администрирование
- Другие меры безопасности достаточны

## Итог

SELinux и AppArmor добавляют мандатный контроль доступа поверх традиционной системы прав Linux. Это дополнительный уровень защиты, который ограничивает возможности процессов независимо от прав пользователя.
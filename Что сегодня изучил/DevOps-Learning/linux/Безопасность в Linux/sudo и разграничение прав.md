Команда `sudo` — это временный пропуск к правам администратора. Сегодня разберёмся, как всё же работает система прав в Linux и как безопасно получать привилегии администратора.

### Почему нельзя работать под root постоянно?

`Root` в Linux — это суперпользователь с неограниченными правами. Работать под `root` постоянно — это как ходить по минному полю:

- Одна опечатка может **уничтожить систему** (`rm -rf /` вместо `rm -rf ./`)
- **Вредоносное ПО** получит **полный контроль** над системой
- **Нет логирования** — непонятно, кто что сделал
- Случайные изменения системных файлов **могут сломать ОС**

Поэтому придумали `sudo` — способ временно получать права `root` только когда это действительно нужно.

### Как работает система прав в Linux

В Linux есть т**ри уровня доступа**:

```scss
┌─────────────────────────────────────────┐
│            Пирамида прав                │
│                                         │
│             root (UID 0)                │ ← Полный доступ
│          ╱────────────────╲             │
│         ╱                  ╲            │
│        ╱     Системные      ╲           │ ← Службы и демоны
│       ╱     пользователи     ╲          │   (apache, mysql)
│      ╱     (UID 1-999)        ╲         │
│     ╱──────────────────────────╲        │
│    ╱     Обычные пользователи   ╲       │ ← Реальные люди
│   ╱          (UID 1000+)         ╲      │   (you, john, alice)
└─────────────────────────────────────────┘
```

Каждый файл и процесс в системе **имеет владельца**. Посмотрим на примере:

```bash
$ ls -la /etc/passwd
-rw-r--r-- 1 root root 2837 Nov 15 10:23 /etc/passwd
            │     │
            │     └── Группа-владелец
            └──────── Пользователь-владелец


$ ps aux | head -3

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.2 169736 11200 ?        Ss   08:30   0:02 /sbin/init
mysql     1053  0.1  1.5 1166876 125440 ?      Ssl  08:30   0:15 /usr/sbin/mysqld
```

### Что такое sudo?

**sudo** (Super User DO) — это программа, которая позволяет выполнять команды **от имени другого пользователя** (обычно `root`).

### Как работает sudo изнутри

Когда вы вводите `sudo команда`, происходит следующее:

```markdown
1. sudo проверяет файл /etc/sudoers
   ↓
2. "Есть ли у этого пользователя права?"
   ↓
3. Запрашивает пароль пользователя (не root)
   ↓
4. Проверяет правильность пароля
   ↓
5. Записывает в лог что и кто делает
   ↓
6. Выполняет команду с правами root
   ↓
7. Возвращает управление пользователю
```

### Настройка sudo: файл /etc/sudoers

Файл `/etc/sudoers` определяет, кто может использовать `sudo` и как. Это критически важный файл, поэтому редактировать его нужно только через `visudo`:

```shell
# НИКОГДА не редактируйте напрямую
$ sudo nano /etc/sudoers  # НЕТ! Опасно!

# Правильный способ
$ sudo visudo
# visudo проверяет синтаксис перед сохранением
```

Основной формат правил в `sudoers`:

```undefined
кто  откуда=(от_чьего_имени)  какие_команды
```

Разберём типичный файл `sudoers`:

```ruby
# Пользователь root может всё
root    ALL=(ALL:ALL) ALL

# Группа admin может всё
%admin  ALL=(ALL) ALL

# Группа sudo может всё (Ubuntu/Debian)
%sudo   ALL=(ALL:ALL) ALL

# Пользователь john может перезагружать Apache
john    ALL=(root) /usr/sbin/apache2ctl restart

# alice может монтировать USB без пароля
alice   ALL=(root) NOPASSWD: /usr/bin/mount, /usr/bin/umount
```

### Практические примеры настройки `sudo`

**Пример 1: Дать пользователю ограниченные права**

Условно у нас есть разработчик, которому нужно **только перезапускать** веб-сервер:

```bash
$ sudo visudo

# Добавляем строку:
webdev ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx

# Теперь webdev может:
$ sudo systemctl restart nginx  ✓
$ sudo systemctl stop nginx     ✗ (не разрешено)
$ sudo apt update              ✗ (не разрешено)
```

**Пример 2: Создать группу с правами `sudo`**

```bash
# Создаём группу developers
$ sudo groupadd developers

# Добавляем пользователей в группу
$ sudo usermod -aG developers alice
$ sudo usermod -aG developers bob

# Настраиваем sudo для группы
$ sudo visudo
# Добавляем:
%developers ALL=(ALL) /usr/bin/docker*, /usr/bin/git

# Теперь alice и bob могут использовать docker и git с sudo
```

**Пример 3: Разрешить выполнение без пароля**

```ruby
# Для скриптов автоматизации иногда нужно sudo без пароля
$ sudo visudo

# Конкретные команды без пароля
backup_user ALL=(root) NOPASSWD: /usr/local/bin/backup.sh

# Все команды без пароля (опасно!)
john ALL=(ALL) NOPASSWD: ALL
```

### Алиасы в `sudoers`

Для удобства можно создавать **алиасы** — группы команд, пользователей или хостов:

```bash
$ sudo visudo

# Алиасы пользователей
User_Alias ADMINS = alice, bob, charlie
User_Alias WEBDEVS = john, jane

# Алиасы команд  
Cmnd_Alias WEB_CMDS = /usr/bin/systemctl restart nginx, \
                      /usr/bin/systemctl reload nginx, \
                      /usr/bin/tail -f /var/log/nginx/*

Cmnd_Alias POWER_CMDS = /usr/sbin/reboot, /usr/sbin/poweroff

# Алиасы хостов
Host_Alias SERVERS = server1, server2, 192.168.1.0/24

# Используем алиасы
ADMINS ALL=(ALL) ALL
WEBDEVS SERVERS=(root) WEB_CMDS
%operators ALL=(root) POWER_CMDS
```

### Полезные опции sudo

```shell
# Выполнить команду от имени другого пользователя
$ sudo -u postgres psql

# Запустить shell с правами root
$ sudo -s           # Использует ваш shell
$ sudo -i           # Логин как root (полное окружение)
$ sudo su           # Старый способ

# Редактировать файл с правами root
$ sudo -e /etc/hosts    # Безопаснее чем sudo nano
$ sudoedit /etc/hosts   # То же самое

# Проверить свои sudo права
$ sudo -l
User alice may run the following commands on server:
    (ALL) ALL
    (root) NOPASSWD: /usr/bin/docker

# Выполнить команду в фоне
$ sudo -b long_running_command

# Проверить синтаксис sudoers
$ sudo visudo -c
/etc/sudoers: parsed OK
```
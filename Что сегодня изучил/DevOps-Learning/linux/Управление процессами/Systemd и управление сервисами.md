## ![](https://ucarecdn.stepik.net/0e8c4ba9-362a-42aa-9516-56daff0cedc8/)

**Systemd** — современная система инициализации и управления сервисами, используемая во всех основных Linux-дистрибутивах. Она отвечает за запуск и остановку служб (сервисов), управление их состоянием, контроль над загрузкой системы, а также мониторинг и журналирование процессов.

### Основные задачи systemd

Systemd выполняет множество задач, среди которых:

- Управление службами и демонами (запуск, остановка, перезапуск).
- Мониторинг состояния сервисов.
- Управление последовательностью загрузки Linux.
- Журналирование работы процессов (с помощью journalctl).
- Планирование периодических задач (systemd timers).
- Управление ресурсами с помощью cgroups.

### Основные команды управления сервисами

Рассмотрим наиболее часто используемые команды systemd.

- Запуск сервиса:
    
    ```bash
    sudo systemctl start имя_сервиса
    ```
    
- Остановка сервиса:
    
    ```bash
    sudo systemctl stop имя_сервиса
    ```
    
- Перезапуск сервиса:
    
    ```bash
    sudo systemctl restart имя_сервиса
    ```
    
- Перезагрузка конфигурации без остановки сервиса:
    
    ```bash
    sudo systemctl reload имя_сервиса
    ```
    
- Проверка текущего состояния сервиса:
    
    ```bash
    sudo systemctl status имя_сервиса
    ```
    

Например, чтобы проверить состояние веб-сервера nginx:

```bash
sudo systemctl status nginx
```

### Управление автозапуском сервисов

Вы можете включить или отключить автоматический запуск сервиса при загрузке системы:

- Включить автозапуск сервиса:
    
    ```bash
    sudo systemctl enable nginx
    ```
    
- Отключить автозапуск сервиса:
    
    ```bash
    sudo systemctl disable nginx
    ```
    
- Проверить, включён ли автозапуск:
    
    ```bash
    sudo systemctl is-enabled nginx
    ```
    

### Просмотр журналов сервисов (journalctl)

Systemd ведёт журнал работы всех сервисов. Просмотреть логи можно командой **journalctl**:

- Просмотреть последние сообщения журнала:
    
    ```bash
    sudo journalctl -xe
    ```
    
- Просмотр журнала конкретного сервиса:
    
    ```bash
    sudo journalctl -u nginx
    ```
    
- Просмотр журнала с выводом в реальном времени:
    
    ```bash
    sudo journalctl -f
    ```
    
- Просмотр журнала с указанием времени:
    
    ```bash
    sudo journalctl --since "2024-05-01 12:00:00"
    ```
    

### Создание собственного сервиса systemd

Вы можете создать свой собственный systemd-сервис, описав его в файле юнита. Рассмотрим простой пример:

Создайте файл сервиса в директории:

```bash
sudo vim /etc/systemd/system/myservice.service
```

Пример содержимого файла:

```ini
[Unit]
Description=My Custom Service
After=network.target

[Service]
Type=simple
ExecStart=/home/user/my_script.sh
Restart=on-failure
Type=oneshot

[Install]
WantedBy=multi-user.target
```

> [Unit]     — описание и зависимости. После изменения unit-файлов выполните: sudo systemctl daemon-reload  
> [Service]  — как запускать/останавливать (ExecStart, Type, Restart и т.д.)  
> [Install]  — как подключать к таргетам (WantedBy=multi-user.target)

Описание параметров:

- `Description` — описание сервиса.
- `After` — указывает, после каких служб запускать сервис (например, после сети).
- `Type=simple` — стандартный сервис, запускаемый командой ExecStart.
- `ExecStart` — команда или скрипт, запускаемый при старте сервиса.
- `Restart=on-failure` — автоматический перезапуск сервиса при сбоях.
- `WantedBy` — определяет, в каком целевом режиме systemd будет запускаться сервис (обычно multi-user.target).

#### Запуск и автозагрузка вашего сервиса:

```bash
sudo systemctl daemon-reload
sudo systemctl start myservice.service
sudo systemctl enable myservice.service
```

### Systemd timers — замена cron

Systemd timers позволяют легко планировать периодические задачи. Для этого нужно создать два файла: сервис и таймер.

**Пример файла сервиса (`/etc/systemd/system/backup.service`):**

```ini
[Unit]
Description=Daily Backup Service

[Service]
Type=oneshot
ExecStart=/home/user/backup.sh
```

**Пример файла таймера (`/etc/systemd/system/backup.timer`):**

```ini
[Unit]
Description=Runs backup daily at midnight

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

Активируйте и запустите таймер:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
```

> ### Подсказки по OnCalendar + список активных таймеров:
> 
> Ежедневно в 02:30: OnCalendar=*-*-* 02:30:00  
> Посмотреть активные таймеры: systemctl list-timers

### Типичные сценарии использования systemd

- Управление веб-серверами (nginx, apache).
- Автоматический запуск и мониторинг баз данных.
- Создание пользовательских сервисов для автоматизации задач.
- Планирование периодических задач через таймеры вместо cron.
- Просмотр и анализ системных журналов.

> ### ⚡ Мини-шпаргалка 
> 
> - Перезапуск: `sudo systemctl restart nginx`
>     
> - Проверить автозапуск: `sudo systemctl is-enabled myapp`
>     
> - Логи сервиса (live): `journalctl -u sshd -f`
>     
> - Секции юнита: `[Unit] [Service] [Install]`
>     
> - Авто-рестарт при сбое: `Restart=on-failure`
>     
> - Перечитать юниты: `sudo systemctl daemon-reload`
>     
> - Включить и запустить: `sudo systemctl enable --now myservice`
>     
> - journalctl часто используемое: `-u`, `-f`, `--since`
>     
> - Таймер ежедневно в 02:30: `OnCalendar=*-*-* 02:30:00`
>     
> - Показать активные таймеры: `systemctl list-timers`
>
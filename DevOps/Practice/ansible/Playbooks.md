# Отчёт по практической работе: Автоматизация DevOps-задач с помощью Ansible
## Цель работы
Написать три Ansible-плейбука для автоматизации типовых задач:
1. Настройка веб-сервера (Nginx + UFW)
2. Деплой веб-приложения (Flask + systemd)
3. Настройка мониторинга (Node Exporter)
---
## Структура проекта
.
├── inventory
│   └── hosts.ini
└── playbooks
    ├── 1-webserver-setup.yml
    ├── 2-app-deploy.yml
    ├── 3-monitoring-setup.yml
    ├── files
    │   └── app.py
    ├── group_vars
    │   └── all.yml
    └── templates
        ├── myapp.service.j2
        ├── node_exporter.service.j2
        └── server.conf.j2


---
## Переменные
### Общие (`group_vars/all.yml`):
```yaml
app_user: appuser
app_directory: /var/www/myapp
domain_name: myapp.local
```

### Локальные:

| Плейбук      | Переменная              | Значение                                     |
| ------------ | ----------------------- | -------------------------------------------- |
| 2-app-deploy | `app_version`           | 1.0                                          |
| 2-app-deploy | `app_port`              | 5000                                         |
| 2-app-deploy | `venv_dir`              | `{{ app_directory }}/venv`                   |
| 3-monitoring | `node_exporter_version` | 1.7.0                                        |
| 3-monitoring | `prometheus_server_ip`  | 192.168.64.100 (свой адрес, не как в задаче) |
## Описание плейбуков

### Плейбук 1: Настройка веб-сервера (`1-webserver-setup.yml`)

**Назначение:** Подготовка сервера к работе веб-приложения.

**Выполняемые задачи:**

- Обновление кэша apt и установка пакетов: `nginx`, `ufw`, `python3-pip`
    
- Создание пользователя `appuser` с домашней директорией
    
- Создание рабочей директории `/var/www/myapp` с правами для `appuser`
    
- Настройка UFW: разрешение портов 80 (HTTP) и 2222 (SSH), включение фаервола
    
- Деплой базового конфига Nginx в `sites-available` с активацией через `sites-enabled`
    
- Удаление default-сайта Nginx
    
- Запуск Nginx и добавление в автозагрузку
    
- Использование handler для перечитывания конфига при изменениях
    

```yml
- name: Настройка веб-сервера
  hosts: webservers
  gather_facts: true
  become: true
  tasks:
   - name: Обновление кеша пакетов
     apt:
      update_cache: yes
      cache_valid_time: 3600
   - name: Установка пакетов
     apt:
      name:
       - nginx
       - ufw
       - python3-pip
      state: present
   - name: Создание пользователя
     user:
      name: "{{ app_user }}"
      createhome: yes
      state: present
   - name: Создание директории "{{ app_directory }}"
     file:
      path: "{{ app_directory }}"
      state: directory
      owner: "{{ app_user }}"
      group: "{{ app_user }}"
      mode: "0755"
   - name: Разрешение 2222 и 80 порта в ufw
     ufw:
      rule: allow
      port: "{{ item }}"
      proto: tcp
     loop:
      - "80"
      - "2222"
   - name: Включение ufw
     ufw:
      state: enabled
   - name: Копирование в sites_available
     template:
      src: templates/server.conf.j2
      dest: "/etc/nginx/sites-available/{{ domain_name }}.conf"
      owner: root
      mode: "0644"
     notify: Перечитывание конфига nginx
   - name: Создание линка в sites-enabled
     file:
      src: "/etc/nginx/sites-available/{{ domain_name }}.conf"
      dest: "/etc/nginx/sites-enabled/{{ domain_name }}.conf"
      state: link
   - name: Удаление default из sites-enabled
     file:
      path: /etc/nginx/sites-enabled/default
      state: absent
   - name: Запуск nginx
     service:
      name: nginx
      state: started
      enabled: true
  handlers:
   - name: Перечитывание конфига nginx
     service:
      name: nginx
      state: reloaded
```
---

### Плейбук 2: Деплой приложения (`2-app-deploy.yml`)

**Назначение:** Развёртывание Flask-приложения с настройкой reverse-proxy.

**Выполняемые задачи:**

- Установка `python3-venv` для виртуального окружения
    
- Копирование `app.py` в `/var/www/myapp/`
    
- Создание и настройка виртуального окружения Python
    
- Установка Flask 2.3.3 в виртуальное окружение
    
- Создание директории `/var/log/myapp` для логов приложения
    
- Деплой systemd unit-файла для управления приложением
    
- Обновление конфига Nginx (добавление `proxy_pass` на `localhost:5000`)
    
- Запуск сервиса `myapp` и добавление в автозагрузку
    
- Проверка доступности приложения через `uri`-модуль
    

```yml
- name: Деплой приложения
  hosts: app_servers
  become: true
  gather_facts: true
  vars:
   app_version: 1.0
   app_port: 5000
   venv_dir: "{{ app_directory }}/venv"
  tasks:
   - name: Установка пакета для виртуального окружения python
     apt:
      name: python3-venv
      state: present
   - name: Копирование файлов приложения
     copy:
      src: files/app.py
      dest: /var/www/myapp/app.py
      owner: "{{ app_user }}"
      group: "{{ app_user }}"
      mode: "0755"
   - name: Создание каталога для python venv
     file:
      path: "{{ venv_dir }}"
      state: directory
      owner: "{{ app_user }}"
      group: "{{ app_user }}"
      mode: "0755"
      recurse: yes
   - name: Создаем виртуальное окружение
     command: python3 -m venv {{ venv_dir }}
     args:
      creates: "{{ venv_dir }}/bin/python"
     become_user: "{{ app_user }}"
   - name: Установка зависимостей
     pip:
      name: Flask\==2.3.3
      virtualenv: "{{ venv_dir }}"
      virtualenv_command: python3 -m venv
      state: present
   - name: Создание каталога для логов
     file:
      path: /var/log/myapp
      state: directory
      owner: "{{ app_user }}"
      group: "{{ app_user }}"
      mode: "0755"
   - name: Копируем systemd unit
     template:
      src: templates/myapp.service.j2
      dest: /etc/systemd/system/myapp.service
      owner: root
      mode: "0644"
     notify: Daemon-reload
   - name: Копирование изменённого конфига nginx(добавление реверса)
     template:
      src: templates/server.conf.j2
      dest: "/etc/nginx/sites-available/{{ domain_name }}.conf"
      owner: root
      mode: "0644"
     notify: Перечитывание конфига nginx
   - name: Запуск myapp.service и добавление в автозапуск
     service:
      name: myapp
      state: started
      enabled: true
   - name: Проверка работы приложения
     uri:
      url: http://myapp.local
      method: GET
      status_code: 200
  handlers:
   - name: Daemon-reload
     systemd:
      name: myapp
      daemon-reload: yes
      state: restarted
   - name: Перечитывание конфига nginx
     service:
      name: nginx
      state: reloaded
```

---

### Плейбук 3: Настройка мониторинга (`3-monitoring-setup.yml`)

**Назначение:** Установка Node Exporter для сбора метрик сервера.

**Выполняемые задачи:**

- Проверка наличия UFW
    
- Создание системного пользователя `node_exporter` (без логина и домашней директории)
    
- Создание директории `/opt/node_exporter`
    
- Скачивание Node Exporter v1.7.0 с проверкой контрольной суммы
    
- Распаковка архива
    
- Деплой systemd unit-файла для Node Exporter
    
- Настройка UFW: открытие порта 9100 только для IP Prometheus-сервера
    
- Запуск сервиса и добавление в автозагрузку
    
- Проверка доступности метрик через `uri`-модуль (`/metrics`)

```yml
- name: Настройка мониторинга
  hosts: monitoring
  gather_facts: true
  become: true
  vars:
   node_exporter_version: 1.7.0
   prometheus_server_ip: 192.168.64.100
  tasks:
   - name: Проверка зависимостей
     apt:
      name: ufw
      state: present
   - name: Создадим пользолвателя node_exporter
     user:
      name: node_exporter
      shell: /bin/false
      system: yes
      createhome: no
      state: present
   - name: Создадим директорию /opt/node_exporter
     file:
      path: /opt/node_exporter
      state: directory
      owner: node_exporter
      group: node_exporter
      mode: "0755"
   - name: Скачивание node_exporter
     get_url:
      url: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
      dest: /opt/node_exporter/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz
      mode: "0644"
      checksum: "sha256:a550cd5c05f760b7934a2d0afad66d2e92e681482f5f57a917465b1fba3b02a6"
   - name: Распаковка архива node_exporter
     unarchive:
      src: /opt/node_exporter/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz
      dest: /opt/node_exporter/
      remote_src: yes
      owner: node_exporter
      group: node_exporter
      extra_opts: [--strip-components=1]
      creates: /opt/node_exporter/node_exporter
   - name: Копирование юнита systemd
     template:
      src: templates/node_exporter.service.j2
      dest: /etc/systemd/system/node_exporter.service
      owner: root
      mode: "0644"
     notify: Daemon-reload
   - name: Открытие порта в ufw
     ufw:
      rule: allow
      proto: tcp
      port: "9100"
      from_ip: "{{ prometheus_server_ip }}"
   - name: Проверка что ufw запущен
     ufw:
      state: enabled
   - name: Запуск сервиса node_exporter и добавление в автозапуск
     service:
      name: node_exporter
      state: started
      enabled: true
   - name: Проверка доступности метрик
     uri:
      url: http://localhost:9100/metrics
      method: GET
      status_code: 200
  handlers:
   - name: Daemon-reload
     systemd:
      name: node_exporter
      daemon-reload: yes
      state: restarted
```
---

## Шаблоны

Шаблон конфига nginx
``` conf
server {
 server_name {{ domain_name }};
 root {{ app_directory }};
 access_log /var/log/nginx/{{ domain_name }}_access.log;
 error_log  /var/log/nginx/{{ domain_name }}_error.log;

 location / {
    proxy_pass http://localhost:{{ app_port }};
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
 }
}
```

Юнит python приложения
```service
[Unit]
Description=Простое Flask приложение, версия {{ app_version }}
After=network.target nginx.service
Wants=nginx.service

[Service]
Type=simple
Environment="PATH={{ venv_dir }}/bin:$PATH"
ExecStart={{ venv_dir }}/bin/python {{ app_directory }}/app.py
Restart=on-failure
RestartSec=5s
User={{ app_user }}
Group={{ app_user }}
WorkingDirectory={{ app_directory }}
NoNewPrivileges=true
StandardOutput=append:/var/log/myapp/app.log
StandardError=append:/var/log/myapp/app_error.log


[Install]
WantedBy=multi-user.target
```

Юнит экспортера
```service
[Unit]
Description=Node exporter version: {{ node_exporter_version }}
After=network.target

[Service]
Type=simple
ExecStart=/opt/node_exporter/node_exporter
User=node_exporter
Group=node_exporter
Restart=always
RestartSec=3s
WorkingDirectory=/opt/node_exporter
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target

```
## Проверка работоспособности

curl -I http://worker_node/

HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Sat, 18 Jul 2026 11:05:53 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 423
Connection: keep-alive

 curl -I curl http://worker_node:9100/metrics
 
HTTP/1.1 200 OK
Content-Type: text/plain; version=0.0.4; charset=utf-8
Date: Sat, 18 Jul 2026 11:06:25 GMT

systemctl status nginx

● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-07-18 08:18:33 UTC; 2h 48min ago
       Docs: man:nginx(8)
   Main PID: 1307 (nginx)
      Tasks: 3 (limit: 3408)
     Memory: 3.9M (peak: 4.4M)
        CPU: 32ms
     CGroup: /system.slice/nginx.service
             ├─1307 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─1309 "nginx: worker process"
             └─1310 "nginx: worker process



systemctl status node_exporter

● node_exporter.service
     Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-07-18 10:47:59 UTC; 19min ago
   Main PID: 7901 (node_exporter)
      Tasks: 4 (limit: 3408)
     Memory: 4.7M (peak: 6.1M)
        CPU: 69ms
     CGroup: /system.slice/node_exporter.service
             └─7901 /opt/node_exporter/node_exporter

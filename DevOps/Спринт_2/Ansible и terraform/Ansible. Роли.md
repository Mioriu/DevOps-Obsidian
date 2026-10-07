## Зачем это нужно?

Без ролей ansible плейбуки превращаются в кашу:

- Один плейбук на 500 строк — не поймешь что делает
- Копипаста одних и тех же задач в разных плейбуках
- Невозможно переиспользовать код
- Нельзя поделиться с другими проектами

**Роли решают это** — структурируют код, делают его переиспользуемым и понятным.

На собеседованиях обязательно спросят про структуру ролей и как их правильно использовать.

## 1. Что такое роль?

**Роль** — это набор задач, переменных, файлов и шаблонов, упакованных в стандартную структуру директорий.

Роль = логически завершенная единица. Примеры:

- Роль для установки nginx
- Роль для настройки firewall
- Роль для деплоя приложения
- Роль для настройки мониторинга

**Преимущества:**

- Переиспользование кода (одну роль можно применить к разным серверам)
- Модульность (роли можно комбинировать)
- Читаемость (понятная структура)
- Тестируемость (роль можно тестировать отдельно)
- Можно выложить в Ansible Galaxy

## 2. Структура роли

Стандартная структура директорий:

codecopy

```
roles/  nginx/                    # имя роли    tasks/                  # задачи (обязательно)      main.yml              # точка входа    handlers/               # обработчики      main.yml    templates/              # jinja2 шаблоны      nginx.conf.j2    files/                  # статические файлы      index.html    vars/                   # переменные (высокий приоритет)      main.yml    defaults/               # переменные по умолчанию (низкий приоритет)      main.yml    meta/                   # метаданные роли (зависимости, автор)      main.yml    tests/                  # тесты роли      test.yml
```

**Обязательная только директория tasks/**, остальное опционально.

### tasks/main.yml

Это точка входа — первый файл который выполняется.

yamlcopy

```yaml
---# Можно писать задачи прямо здесь- name: Install nginx  apt:    name: nginx    state: present # Или разбить на несколько файлов и импортировать- import_tasks: install.yml- import_tasks: configure.yml- import_tasks: start.yml
```

### handlers/main.yml

Обработчики — задачи которые выполняются только если что-то изменилось.

yamlcopy

```yaml
---- name: restart nginx  service:    name: nginx    state: restarted - name: reload nginx  service:    name: nginx    state: reloaded
```

Вызов из задачи:

yamlcopy

```yaml
- name: Copy nginx config  template:    src: nginx.conf.j2    dest: /etc/nginx/nginx.conf  notify: restart nginx    # вызовет handler только если конфиг изменился
```

### templates/

Jinja2 шаблоны с переменными.

jinja2copy

```jinja2
# nginx.conf.j2worker_processes {{ nginx_workers }};server {    listen {{ nginx_port }};    server_name {{ server_name }};}
```

### files/

Статические файлы которые копируются как есть (без подстановки переменных).

### defaults/ vs vars/

**defaults/main.yml** — переменные по умолчанию (низкий приоритет):

yamlcopy

```yaml
nginx_port: 80nginx_workers: 2
```

**vars/main.yml** — переменные роли (высокий приоритет):

nginx_user: www-data

**Разница в приоритете:**

- `defaults/` — самый низкий приоритет, легко переопределить извне
- `vars/` — высокий приоритет, сложно переопределить

**Практика:** Используй `defaults/` для всего что пользователь может захотеть изменить.

### meta/main.yml

Метаданные роли: зависимости, минимальная версия ansible, автор.

yamlcopy

```yaml
---dependencies:  - role: common           # эта роль зависит от роли common  - role: firewall    vars:      port: 80 galaxy_info:  author: Your Name  description: Nginx web server  min_ansible_version: 2.9  platforms:    - name: Ubuntu      versions:        - focal        - jammy
```

## 3. Как использовать роли

### В playbook

yamlcopy

```yaml
---- hosts: webservers  roles:    - nginx                          # простое подключение    - role: postgresql               # с переопределением переменных      vars:        postgres_version: 14    - role: app      become: yes                    # с дополнительными параметрами
```

### Через tasks

yamlcopy

```yaml
---- hosts: webservers  tasks:    - name: Apply nginx role      include_role:        name: nginx      vars:        nginx_port: 8080
```

### Где ansible ищет роли?

По умолчанию:

1. `./roles/` — директория roles рядом с playbook
2. `/etc/ansible/roles/` — системная директория
3. Путь из `ansible.cfg`:

inicopy

```ini
[defaults]roles_path = ./roles:/usr/share/ansible/roles
```

## 4. Переменные в ролях

### Приоритет переменных (от низкого к высокому)

1. `role defaults` (defaults/main.yml) — **самый низкий**
2. inventory vars
3. playbook vars
4. `role vars` (vars/main.yml)
5. extra vars (`-e` в командной строке) — **самый высокий**

Пример:

yamlcopy

```yaml
# defaults/main.ymlnginx_port: 80 # playbook.yml- hosts: web  roles:    - role: nginx      vars:        nginx_port: 8080    # переопределит defaults # командная строкаansible-playbook -e "nginx_port=9000"  # переопределит все
```

**Правило:** В `defaults/` пиши переменные которые пользователь ДОЛЖЕН переопределять. В `vars/` — константы роли.

## 5. Зависимости между ролями

Роль может зависеть от других ролей через `meta/main.yml`:

yamlcopy

```yaml
# roles/app/meta/main.yml---dependencies:  - role: common           # сначала выполнится common  - role: nginx            # потом nginx    vars:                  # с этими переменными      nginx_port: 8080
```

**Важно:** Зависимости выполняются ПЕРЕД задачами роли.

Порядок выполнения:

1. Зависимости роли (common, nginx)
2. Задачи роли (app)

## 6. Ansible Galaxy

**Ansible Galaxy** — публичный репозиторий готовых ролей.

### Установить роль из Galaxy

bashcopy

```bash
# Установить рольansible-galaxy install geerlingguy.nginx # Установить в конкретную директориюansible-galaxy install geerlingguy.nginx -p ./roles # Установить из requirements.ymlansible-galaxy install -r requirements.yml
```

### requirements.yml

Файл с зависимостями проекта:

yamlcopy

```yaml
---# Из Galaxy- name: geerlingguy.nginx  version: 3.1.4 # Из Git- src: https://github.com/user/role.git  name: custom_role  version: main # Из локального пути- src: /path/to/role  name: local_role
```

Установка:

ansible-galaxy install -r requirements.yml

### Создать свою роль

bashcopy

```bash
# Создать структуру ролиansible-galaxy init my_role # Создаст:# my_role/#   tasks/main.yml#   handlers/main.yml#   ...
```

## 7. Best Practices

### Именование

- Используй змеиный_регистр: `web_server`, `db_backup`
- Префикс переменных именем роли: `nginx_port`, `postgres_version`
- Избегай конфликта имен между ролями

### Структура

- Одна роль = одна ответственность (nginx, postgres, но не nginx_postgres)
- Разбивай большие tasks на несколько файлов
- Используй `defaults/` для настраиваемых параметров

### Переменные

yamlcopy

```yaml
# Плохо - хардкод- name: Install package  apt:    name: nginx # Хорошо - через переменную- name: Install package  apt:    name: "{{ package_name }}"
```

### Идемпотентность

Роль должна быть идемпотентной — можно запускать много раз без побочных эффектов.

yamlcopy

```yaml
# Плохо - создаст файл каждый раз- shell: echo "test" >> /tmp/file # Хорошо - проверит существование- copy:    content: "test"    dest: /tmp/file    force: no
```

### Тестирование

yamlcopy

```yaml
# roles/nginx/tests/test.yml---- hosts: localhost  roles:    - nginx
```

Запуск:

ansible-playbook roles/nginx/tests/test.yml --connection=local

Или через Molecule для полноценного тестирования.

## 8. Типичные ошибки

### Ошибка 1: Роль слишком большая

Плохо: Роль делает установку nginx + настройку php + деплой приложения

Хорошо: Три отдельные роли (nginx, php, app)

### Ошибка 2: Хардкод значений

Плохо:

yamlcopy

```yaml
- name: Copy config  template:    src: nginx.conf.j2    dest: /etc/nginx/nginx.conf    owner: nginx
```

Хорошо:

yamlcopy

```yaml
- name: Copy config  template:    src: nginx.conf.j2    dest: "{{ nginx_config_path }}"    owner: "{{ nginx_user }}"
```

### Ошибка 3: Игнорирование различий ОС

Плохо: Роль работает только на Ubuntu

Хорошо:

yamlcopy

```yaml
- name: Install nginx (Debian)  apt:    name: nginx  when: ansible_os_family == "Debian" - name: Install nginx (RedHat)  yum:    name: nginx  when: ansible_os_family == "RedHat"
```

## 9. Вопросы на собеседовании

**Что такое роль в Ansible?**

Роль — это набор задач, переменных, файлов и шаблонов в стандартной структуре директорий. Позволяет переиспользовать код и структурировать playbook.

**В чем разница между defaults и vars?**

`defaults/` — переменные с низким приоритетом, легко переопределить. Используются для параметров которые пользователь может настроить.

`vars/` — переменные с высоким приоритетом, сложно переопределить. Используются для констант роли.

**Как работают зависимости ролей?**

Зависимости указываются в `meta/main.yml`. Они выполняются ПЕРЕД задачами самой роли. Порядок: зависимости → задачи роли.

**Зачем нужны handlers?**

Handlers выполняются только если задача изменила состояние системы (changed). Используются для перезапуска сервисов, перечитывания конфигов и т.д.

**Что такое Ansible Galaxy?**

Публичный репозиторий готовых ролей. Можно устанавливать чужие роли и публиковать свои.

**В каком порядке выполняются роли в playbook?**

В том порядке как они указаны. Для каждой роли сначала выполняются зависимости из `meta/main.yml`, затем задачи из `tasks/main.yml`.

**Как переопределить переменную роли?**

Несколько способов:

- В playbook через `vars:`
- В inventory
- Через extra vars: `ansible-playbook -e "var=value"`
- Переопределяя defaults создав `group_vars/` или `host_vars/`
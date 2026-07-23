Модули — это то, что делает реальную работу. В tasks ты вызываешь модули.

## 1. Управление пакетами

### apt (Debian/Ubuntu)

yamlcopy

```yaml
- name: Install nginx  apt:    name: nginx    state: present    update_cache: yes - name: Install multiple packages  apt:    name:      - nginx      - postgresql      - redis    state: present
```

### yum/dnf (RHEL/CentOS)

yamlcopy

```yaml
- name: Install nginx  yum:    name: nginx    state: present - name: Remove package  yum:    name: httpd    state: absent
```

**state:**

- `present` — установить если нет
- `absent` — удалить если есть
- `latest` — установить последнюю версию

## 2. Управление файлами

### copy

Копирует файл с control node на remote host:

yamlcopy

```yaml
- name: Copy file  copy:    src: /local/path/file.txt    dest: /remote/path/file.txt    owner: root    group: root    mode: '0644' - name: Copy with inline content  copy:    content: "Hello world\n"    dest: /tmp/hello.txt
```

### file

Управление файлами/директориями:

yamlcopy

```yaml
- name: Create directory  file:    path: /opt/app    state: directory    owner: www-data    mode: '0755' - name: Create symlink  file:    src: /opt/app/current    dest: /var/www/app    state: link - name: Delete file  file:    path: /tmp/file.txt    state: absent
```

**state:**

- `file` — проверить что файл существует
- `directory` — создать директорию
- `link` — создать symlink
- `absent` — удалить

### template

Копирует Jinja2 шаблон с подстановкой переменных:

yamlcopy

```yaml
- name: Copy nginx config from template  template:    src: nginx.conf.j2    dest: /etc/nginx/nginx.conf    owner: root    mode: '0644'  notify: restart nginx
```

### fetch

Копирует файл С remote host НА control node (обратно copy):

yamlcopy

```yaml
- name: Fetch log file  fetch:    src: /var/log/app.log    dest: /local/logs/    flat: yes
```

## 3. Управление сервисами

### service / systemd

yamlcopy

```yaml
- name: Start nginx  service:    name: nginx    state: started    enabled: yes - name: Restart service  systemd:    name: postgresql    state: restarted    daemon_reload: yes
```

**state:**

- `started` — запустить если не запущен
- `stopped` — остановить
- `restarted` — перезапустить
- `reloaded` — reload конфига

**enabled:**

- `yes` — добавить в автозапуск
- `no` — убрать из автозапуска

## 4. Команды и скрипты

### command

Выполняет команду (без shell):

yamlcopy

```yaml
- name: Run command  command: /usr/bin/myapp --config /etc/app.conf  args:    chdir: /opt/app
```

**Важно:** Не поддерживает pipes, редиректы, переменные окружения.

### shell

Выполняет через shell (bash):

yamlcopy

```yaml
- name: Run shell command  shell: ps aux | grep nginx | wc -l - name: Complex command  shell: |    export VAR=value    echo $VAR > /tmp/file    cat /tmp/file | grep value
```

**Используй shell когда нужны:** pipes, редиректы, переменные окружения.

### script

Копирует скрипт и выполняет:

yamlcopy

```yaml
- name: Run local script on remote  script: /local/path/script.sh
```

## 5. Управление пользователями

### user

yamlcopy

```yaml
- name: Create user  user:    name: deploy    shell: /bin/bash    groups: sudo    append: yes    state: present - name: Add SSH key  user:    name: deploy    generate_ssh_key: yes    ssh_key_bits: 2048
```

### group

yamlcopy

```yaml
- name: Create group  group:    name: developers    state: present
```

## 6. Управление Git

### git

yamlcopy

```yaml
- name: Clone repository  git:    repo: https://github.com/user/repo.git    dest: /opt/app    version: main    force: yes - name: Update repo  git:    repo: https://github.com/user/repo.git    dest: /opt/app    update: yes
```

## 7. Управление архивами

### unarchive

yamlcopy

```yaml
- name: Extract tar.gz  unarchive:    src: /tmp/archive.tar.gz    dest: /opt/app    remote_src: yes - name: Extract from control node  unarchive:    src: /local/archive.tar.gz    dest: /opt/app    remote_src: no
```

**remote_src:**

- `yes` — архив уже на remote host
- `no` — копировать с control node

## 8. Работа с URL

### get_url

Скачать файл по URL:

yamlcopy

```yaml
- name: Download file  get_url:    url: https://example.com/file.tar.gz    dest: /tmp/file.tar.gz    mode: '0644'    checksum: sha256:xxxxx
```

### uri

HTTP запросы:

yamlcopy

```yaml
- name: Check API  uri:    url: https://api.example.com/health    method: GET    status_code: 200 - name: POST request  uri:    url: https://api.example.com/deploy    method: POST    body_format: json    body:      version: "1.2.3"
```

## 9. Управление переменными

### set_fact

Установить переменную:

yamlcopy

```yaml
- name: Set fact  set_fact:    app_version: "1.2.3"    deploy_user: "deploy" - name: Use fact  debug:    msg: "Deploying version {{ app_version }}"
```

### debug

Вывод для отладки:

yamlcopy

```yaml
- name: Print variable  debug:    msg: "Variable value: {{ my_var }}" - name: Print all variables  debug:    var: hostvars[inventory_hostname]
```

## 10. Управление Docker

### docker_container

yamlcopy

```yaml
- name: Run container  docker_container:    name: nginx    image: nginx:latest    state: started    ports:      - "80:80"    volumes:      - /host/path:/container/path    env:      ENV_VAR: value
```

### docker_image

yamlcopy

```yaml
- name: Pull image  docker_image:    name: nginx:latest    source: pull
```

## 11. Линтинг и проверки

### assert

Проверка условий:

yamlcopy

```yaml
- name: Check variable  assert:    that:      - ansible_os_family == "Debian"      - nginx_port > 1024    fail_msg: "OS must be Debian and port > 1024"
```

### stat

Проверка существования файла:

yamlcopy

```yaml
- name: Check if file exists  stat:    path: /etc/nginx/nginx.conf  register: nginx_conf - name: Do something if exists  debug:    msg: "Config exists"  when: nginx_conf.stat.exists
```

## 12. Работа с линиями в файлах

### lineinfile

Добавить/изменить строку в файле:

yamlcopy

```yaml
- name: Add line to file  lineinfile:    path: /etc/hosts    line: "192.168.1.10 server.local"    state: present - name: Replace line  lineinfile:    path: /etc/ssh/sshd_config    regexp: '^PermitRootLogin'    line: 'PermitRootLogin no'
```

### blockinfile

Добавить блок строк:

yamlcopy

```yaml
- name: Add block to file  blockinfile:    path: /etc/hosts    block: |      192.168.1.10 server1.local      192.168.1.11 server2.local    marker: "# {mark} ANSIBLE MANAGED BLOCK"
```

## Самые важные для собеседований:

1. **apt/yum** — установка пакетов
2. **copy/template** — работа с файлами
3. **service/systemd** — управление сервисами
4. **command/shell** — выполнение команд
5. **user/group** — управление пользователями
6. **file** — создание директорий, symlinks
7. **git** — работа с репозиториями
8. **debug** — отладка

**На собесе спросят:**

**"В чем разница между copy и template?"**

`copy` копирует файл как есть. `template` использует Jinja2 шаблон с подстановкой переменных.

**"В чем разница между command и shell?"**

`command` выполняет команду напрямую без shell — безопаснее, но нет pipes/редиректов. `shell` выполняет через bash — поддерживает pipes, переменные окружения, но менее безопасен.

**"Зачем notify в template?"**

`notify` вызывает handler только если шаблон изменился. Используется чтобы не перезапускать сервис если конфиг не поменялся.
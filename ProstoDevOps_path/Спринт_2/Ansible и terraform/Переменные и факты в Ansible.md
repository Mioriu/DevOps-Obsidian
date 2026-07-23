Так, плейбуки писать мы научились, задачи описывать умеем. Но вот проблема — пока что мы хардкодим всё подряд. Имя пакета, имя пользователя, порт, путь к конфигу — всё прибито гвоздями. А теперь представьте: у вас три окружения — dev, staging, prod. На dev nginx слушает порт 8080, на staging — 80, на prod — 443. И что, под каждое окружение свой playbook писать? Конечно нет. Для этого и существуют переменные.

Переменные в Ansible — это, по сути, способ не хардкодить значения, а вынести их отдельно и подставлять в нужные места. Звучит банально, но тема на самом деле глубже, чем кажется, потому что в Ansible переменные можно задать штук двадцатью разными способами, и у каждого свой приоритет.

## vars — переменные прямо в playbook'е

Самый простой способ задать переменные — прямо в play через `vars`:

yamlcopy

```yaml
---- name: Настройка веб-сервера  hosts: webservers  become: true   vars:    http_port: 80    app_name: myapp    app_user: deploy    packages:      - nginx      - curl      - htop   tasks:    - name: Установить пакеты      apt:        name: "{{ packages }}"        state: present     - name: Создать пользователя для приложения      user:        name: "{{ app_user }}"        shell: /bin/bash     - name: Показать настройки      debug:        msg: "Приложение {{ app_name }} будет слушать порт {{ http_port }}"
```

Тут всё просто. В секции `vars` задаём переменные, а потом используем их через двойные фигурные скобки `{{ }}`. Это синтаксис Jinja2 — шаблонизатора, который Ansible использует под капотом.

Один важный момент, если значение начинается с `{{ }}`, его нужно брать в кавычки:

yamlcopy

```yaml
    # ПРАВИЛЬНО    - name: Создать пользователя      user:        name: "{{ app_user }}"     # НЕПРАВИЛЬНО — Ansible упадёт с ошибкой    - name: Создать пользователя      user:        name: {{ app_user }}
```

Без кавычек YAML-парсер думает, что фигурные скобки — это начало словаря, и всё ломается. Если переменная стоит в середине строки — можно без кавычек, но лучше просто всегда ставить кавычки и не думать об этом.

## vars_files — переменные в отдельном файле

Когда переменных становится много, тащить их все в плейбуке неудобно. Файл раздувается, становится сложно читать. Поэтому переменные можно вынести в отдельный файл:

yamlcopy

```yaml
---- name: Настройка веб-сервера  hosts: webservers  become: true  vars_files:    - vars/main.yml    - vars/secrets.yml   tasks:    - name: Установить nginx      apt:        name: nginx        state: present
```

А в файле `vars/main.yml`:

yamlcopy

```yaml
---http_port: 80app_name: myappapp_user: deployapp_dir: /opt/myappmax_connections: 1024
```

Вот так. Плейбук остаётся чистым и компактным, а все настройки лежат отдельно. Плюс, заметьте, мы подключили два файла — `main.yml` и `secrets.yml`. Это удобно, когда хотите отделить обычные переменные от секретов. `secrets.yml` потом можно зашифровать через Ansible Vault, но это тема отдельного материала.

## host_vars и group_vars — переменные для хостов и групп

На практике это используется постоянно. Суть простая: вы можете задать переменные, которые будут автоматически применяться к конкретному хосту или к целой группе хостов.

Для этого нужно создать специальные директории `host_vars` и `group_vars` рядом с inventory-файлом или плейбуком:

![Ansible переменные: group_vars + host_vars](https://offers.prostodevops.ru/diagrams/ansible/ansible-vars-tree.png)Ansible переменные: group_vars + host_vars

### group_vars

Допустим, у вас в inventory есть группа `webservers`. Создаёте файл `group_vars/webservers.yml`:

yamlcopy

```yaml
---http_port: 80nginx_worker_processes: 4app_root: /var/www/html
```

И всё — эти переменные автоматически будут доступны для всех хостов из группы `webservers`. Не нужно ничего подключать, не нужно писать `vars_files`. Ansible сам увидит эту папку и подхватит.

Есть опять же специальная группа `all` — она включает вообще все хосты из inventory. Соответственно, `group_vars/all.yml` — это переменные для всех серверов:

yamlcopy

```yaml
---# group_vars/all.ymlansible_python_interpreter: /usr/bin/python3ntp_server: ntp.company.localdns_servers:  - 8.8.8.8  - 8.8.4.4
```

### host_vars

То же самое, только для конкретного хоста. Файл `host_vars/server1.yml`:

yamlcopy

```yaml
---http_port: 8080nginx_worker_processes: 8custom_domain: server1.company.com
```

Эти переменные будут доступны только для `server1`. И если у `server1` в `host_vars` задан `http_port: 8080`, а в `group_vars/webservers.yml` — `http_port: 80`, то для `server1` будет 8080. Потому что `host_vars` имеет более высокий приоритет, чем `group_vars`. И вот тут мы подходим к самой, наверное, запутанной теме в переменных.

## Приоритеты переменных (variable precedence)

В Ansible переменные можно задать кучей разных способов, и все они имеют разный приоритет. Когда одна и та же переменная задана в нескольких местах — кто побеждает? Вот тут людей на собесах любят гонять

Полный список приоритетов в Ansible — это 22 уровня. Двадцать два. Учить все наизусть — бессмысленно, даже не пытайтесь. Но вот основные уровни, которые нужно понимать, от низкого к высокому:

1. **Defaults роли** (`roles/myrole/defaults/main.yml`) — самый низкий приоритет. Легко перебиваются чем угодно. Это как значения по умолчанию.
    
2. **group_vars/all** — переменные для всех хостов.
    
3. **group_vars/конкретная_группа** — переменные для конкретной группы.
    
4. **host_vars** — переменные для конкретного хоста. Перебивают group_vars.
    
5. **vars в play** — то, что мы писали через `vars:` в плейбуке.
    
6. **vars в task** — переменные на уровне конкретной задачи.
    
7. **Переменные из командной строки** (`-e` или `--extra-vars`) — самый высокий приоритет. Перебивают вообще всё.
    

На практике логика такая: чем конкретнее место, где задана переменная, тем выше её приоритет. Defaults роли — самые общие, их перебивает всё. Командная строка — самая конкретная, она перебивает всех.

Давайте на примере. Допустим, у нас `http_port` задан в трёх местах:

yamlcopy

```yaml
# group_vars/webservers.ymlhttp_port: 80 # host_vars/server1.ymlhttp_port: 8080 # запуск из командной строкиansible-playbook playbook.yml -e "http_port=9090"
```

Для server1 значение будет: — Если запустить без `-e` — 8080 (host_vars перебил group_vars). — Если запустить с `-e "http_port=9090"` — 9090 (командная строка перебивает всё). — Для server2 (у которого нет host_vars) без `-e` — 80 (group_vars).

### Extra vars

Вот эти `-e` или `--extra-vars` — это, по сути, последний аргумент. Самый высокий приоритет. Перебивает вообще всё. Используется, когда нужно прямо здесь и сейчас задать значение, не трогая файлы:

ansible-playbook playbook.yml -e "app_version=2.1.0 deploy_env=production"

Или можно передать файл:

ansible-playbook playbook.yml -e "@vars/release.yml"

На практике extra vars часто используются в CI/CD. В пайплайне передаёте версию приложения, окружение и так далее — и плейбук их использует.

## Факты — информация о серверах

Теперь переходим к фактам. Факты — это переменные, которые Ansible автоматически собирает с серверов при подключении. Информация об операционной системе, количество ядер, объём памяти, IP-адреса, диски, сетевые интерфейсы — всё это Ansible узнаёт сам и делает доступным через переменные.

Когда мы запускаем плейбук, первая задача — `Gathering Facts`? Вот это оно. Ansible подключается к серверу, запускает модуль `setup` и собирает кучу информации.

### Модуль setup

Давайте посмотрим, что конкретно собирает Ansible. Запустите ad-hoc:

ansible server1 -m setup

Вывалится огромная стена JSON. Там сотни переменных. Вот самые полезные из них:

bashcopy

```bash
# только информация об ОСansible server1 -m setup -a "filter=ansible_distribution*" # только информация о памятиansible server1 -m setup -a "filter=ansible_mem*" # только IP-адресаansible server1 -m setup -a "filter=ansible_default_ipv4" # только информация о дискахansible server1 -m setup -a "filter=ansible_mounts"
```

Параметр `filter` позволяет отфильтровать только то, что нужно, чтобы не утонуть в этой горе данных.

### Использование фактов в плейбуках

Факты используются точно так же, как обычные переменные — через двойные фигурные скобки:

yamlcopy

```yaml
---- name: Пример использования фактов  hosts: all  become: true   tasks:    - name: Информация о сервере      debug:        msg: |          Хост: {{ ansible_hostname }}          ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}          Ядер: {{ ansible_processor_cores }}          Память: {{ ansible_memtotal_mb }} MB          IP: {{ ansible_default_ipv4.address }}          Диск /: {{ ansible_mounts | selectattr('mount', 'equalto', '/') | map(attribute='size_total') | first | int / 1073741824 | round(1) }} GB     - name: Установить пакет в зависимости от ОС      apt:        name: nginx        state: present      when: ansible_os_family == "Debian"     - name: Установить пакет на RedHat      yum:        name: nginx        state: present      when: ansible_os_family == "RedHat"
```

Вот обратите внимание на `when: ansible_os_family == "Debian"`. Мы используем факт, чтобы определить, какой модуль вызывать — `apt` или `yum`. Это реальный кейс. Если у вас в inventory намешаны Ubuntu и CentOS — вы не можете вызывать `apt` на всех, CentOS упадёт. А через факты вы определяете тип ОС и действуете соответственно.

### Часто используемые факты

Вот список фактов, которые на практике используются чаще всего:

— `ansible_hostname` — имя хоста. — `ansible_fqdn` — полное доменное имя. — `ansible_default_ipv4.address` — основной IPv4-адрес. — `ansible_distribution` — название дистрибутива (Ubuntu, CentOS и т.д.). — `ansible_distribution_version` — версия дистрибутива (22.04, 9 и т.д.). — `ansible_os_family` — семейство ОС (Debian, RedHat, Suse и т.д.). — `ansible_memtotal_mb` — общий объём оперативки в мегабайтах. — `ansible_processor_cores` — количество ядер. — `ansible_mounts` — информация о смонтированных дисках. — `ansible_env` — переменные окружения.

### Отключение сбора фактов

Если вам факты не нужны — отключайте. На больших inventory это заметно ускоряет выполнение, потому что Ansible не тратит время на подключение к каждому серверу и сбор информации:

yamlcopy

```yaml
- name: Простая задача без фактов  hosts: all  gather_facts: false   tasks:    - name: Просто пингуем      ping:
```

Но имейте в виду — если потом в задачах попытаетесь обратиться к `ansible_hostname` или любому другому факту, оно упадёт, потому что факты не были собраны.

Можно также собрать факты вручную в нужный момент:

yamlcopy

```yaml
  tasks:    - name: Сделать что-то без фактов      ping:     - name: Теперь собрать факты      setup:     - name: Использовать факты      debug:        msg: "IP: {{ ansible_default_ipv4.address }}"
```

## Кастомные факты

Помимо стандартных фактов, которые Ansible собирает автоматически, вы можете создать свои собственные. Кастомные факты — это файлы, которые лежат на целевом сервере в директории `/etc/ansible/facts.d/`. Ansible при сборе фактов проверяет эту папку и подтягивает всё, что там найдёт.

Кастомные факты бывают двух типов — статические (файлы `.fact` в формате INI или JSON) и динамические (исполняемые скрипты).

### Статический факт

Допустим, вы хотите, чтобы на каждом сервере был факт о том, какое это приложение и какая версия. Создаём файл на сервере:

inicopy

```ini
; /etc/ansible/facts.d/app.fact[general]app_name = myappapp_version = 2.1.0environment = production
```

Или можно через playbook это раскинуть на все серверы:

yamlcopy

```yaml
---- name: Настроить кастомные факты  hosts: all  become: true   tasks:    - name: Создать директорию для фактов      file:        path: /etc/ansible/facts.d        state: directory        owner: root        group: root        mode: "0755"     - name: Создать файл с кастомным фактом      copy:        content: |          [general]          app_name = myapp          app_version = 2.1.0          environment = production        dest: /etc/ansible/facts.d/app.fact        owner: root        group: root        mode: "0644"     - name: Перечитать факты      setup:     - name: Показать кастомный факт      debug:        var: ansible_local.app.general
```

Обратите внимание — кастомные факты доступны через `ansible_local`. То есть, `ansible_local.app.general.app_name` вернёт `myapp`. Это важно запомнить: стандартные факты — `ansible_*`, кастомные — `ansible_local.*`.

## Переменные из register

Ещё один способ создать переменную — сохранить результат выполнения задачи. Для этого используется `register`:

yamlcopy

```yaml
  tasks:    - name: Проверить версию nginx      command: nginx -v      register: nginx_result      changed_when: false     - name: Показать версию      debug:        msg: "Nginx: {{ nginx_result.stderr }}"     - name: Показать код возврата      debug:        msg: "Код возврата: {{ nginx_result.rc }}"
```

`register: nginx_result` — сохраняет весь результат задачи в переменную `nginx_result`. Потом эту переменную можно использовать как угодно — показать через debug, использовать в `when` для условий и так далее.

Что лежит в зарегистрированной переменной: — `nginx_result.stdout` — стандартный вывод. — `nginx_result.stderr` — вывод ошибок (nginx -v, кстати, пишет версию именно в stderr, это его особенность). — `nginx_result.rc` — код возврата (0 — успех). — `nginx_result.changed` — изменилось ли что-то.

`changed_when: false` — это чтобы Ansible не считал эту задачу как "что-то изменившую". Потому что мы просто посмотрели версию, ничего не меняли.

## Переменные в inventory

Переменные можно задавать и прямо в inventory-файле. Мы уже видели это:

inicopy

```ini
[webservers]server1 ansible_host=10.0.0.1 http_port=80server2 ansible_host=10.0.0.2 http_port=8080 [webservers:vars]nginx_worker_processes=4app_root=/var/www/html
```

Переменные после имени хоста — это `host_vars`, только заданные в inventory. `[webservers:vars]` — это `group_vars`, только в inventory. По приоритету они работают так же — хостовые перебивают групповые.

Но на практике, когда переменных больше двух-трёх, лучше использовать директории `host_vars/` и `group_vars/` вместо того, чтобы пихать всё в inventory. Inventory должен описывать структуру — какие серверы есть и в какие группы входят. А настройки — отдельно. Так гораздо чище и читаемее.

## Типичная структура проекта с переменными

Вот как выглядит нормально организованный проект с переменными:

![Ansible полный layout с Vault: group_vars + host_vars + vars](https://offers.prostodevops.ru/diagrams/ansible/ansible-vars-full.png)Ansible полный layout с Vault: group_vars + host_vars + vars

Это не единственно правильная структура, но такой подход — это, в принципе, стандарт, который вы увидите в большинстве проектов.
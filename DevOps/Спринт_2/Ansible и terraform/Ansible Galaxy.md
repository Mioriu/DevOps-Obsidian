# Ansible Galaxy

В прошлом материале мы научились создавать свои роли. Но вот в чём дело — огромное количество задач, которые вы будете решать через Ansible, уже кто-то решил до вас. Установить nginx, настроить PostgreSQL, развернуть Docker, настроить файрвол — для всего этого есть готовые роли, которые написаны, протестированы и поддерживаются сообществом. И лежат они в Ansible Galaxy.

Ansible Galaxy — это, по сути, маркетплейс ролей и коллекций. Заходишь, ищешь нужное, ставишь одной командой, подключаешь в плейбук. Но тут, как и везде, есть нюансы — какие роли брать, каким авторам доверять и когда всё-таки лучше написать своё.

## Что такое Galaxy и как искать

Веб-интерфейс Galaxy живёт на [galaxy.ansible.com](https://galaxy.ansible.com/). Там можно искать роли и коллекции через поиск, фильтровать по платформе, по категории, смотреть рейтинги и количество скачиваний.

Но честно говоря, через веб-интерфейс мало кто ищет. Обычно либо гуглят "ansible role nginx", либо ищут прямо из командной строки:

ansible-galaxy search nginx

Выдаст список ролей, связанных с nginx. Можно отфильтровать по платформе:

ansible-galaxy search nginx --platforms Ubuntu

Посмотреть информацию о конкретной роли:

ansible-galaxy info geerlingguy.nginx

Тут покажет описание, поддерживаемые платформы, зависимости, ссылку на репозиторий.

## Установка ролей

Установить роль из Galaxy — одна команда:

ansible-galaxy install geerlingguy.nginx

Роль скачается и положится в `~/.ansible/roles/`, можно подключать в плейбук:

yamlcopy

```yaml
---- name: Настройка веб-сервера  hosts: webservers  become: true  roles:    - geerlingguy.nginx
```

Можно указать, куда поставить:

ansible-galaxy install geerlingguy.nginx -p ./roles/

Флаг `-p` задаёт путь. Так роль ляжет прямо в папку `roles/` вашего проекта, рядом с вашими собственными ролями.

## requirements.yml — управление зависимостями

Когда у вас в проекте используется пять-десять ролей из Galaxy, ставить их по одной руками — неудобно. Плюс коллега склонирует репозиторий и не будет знать, какие роли нужны. Для этого есть `requirements.yml` — файл, в котором перечислены все внешние роли и коллекции.

yamlcopy

```yaml
# requirements.yml---roles:  - name: geerlingguy.nginx    version: "3.1.0"   - name: geerlingguy.postgresql    version: "3.4.0"   - name: geerlingguy.docker    version: "7.1.0"   - name: geerlingguy.node_exporter    version: "2.1.0"
```

Установить все роли из файла:

ansible-galaxy install -r requirements.yml

### Установка ролей из гита

Galaxy — не единственный источник. Можно ставить роли напрямую из гит-репозитория:

yamlcopy

```yaml
# requirements.yml---roles:  - name: nginx    src: https://github.com/company/ansible-role-nginx.git    version: v2.0.0    scm: git   - name: internal-monitoring    src: git@gitlab.company.local:devops/ansible-role-monitoring.git    version: main    scm: git
```

Это полезно, когда у вас в компании есть свои внутренние роли на гитлабе, которых нет в публичном Galaxy. Указываете URL репозитория, ветку или тег — и Ansible скачает.

### Коллекции в requirements.yml

Помимо ролей, в Galaxy есть коллекции — это более крупные пакеты, которые включают роли, модули, плагины. Их тоже можно прописать в `requirements.yml`:

yamlcopy

```yaml
# requirements.yml---roles:  - name: geerlingguy.nginx    version: "3.1.0" collections:  - name: community.general    version: "8.0.0"  - name: community.docker    version: "3.4.0"  - name: community.postgresql    version: "3.2.0"
```

Устанавливаются так же:

bashcopy

```bash
ansible-galaxy install -r requirements.ymlansible-galaxy collection install -r requirements.yml
```

Коллекции — это, по сути, следующий этап развития Galaxy. Многие модули, которые раньше были встроены в Ansible, теперь живут в коллекциях. Например, модули для Docker — в `community.docker`, для PostgreSQL — в `community.postgresql`. Если Ansible говорит, что модуль не найден — скорее всего, нужно доставить коллекцию.
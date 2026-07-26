Подготовка к задаче:
**1. Создать новый проект в GitLab:**

- Название: `pipeline-1-webapp`
- Visibility: Private
- Initialize with README

**2. Настроить переменные окружения (Settings → CI/CD → Variables):**

- `CI_REGISTRY_USER` - username для Container Registry (тип: default)
- `CI_REGISTRY_PASSWORD` - password для Registry (тип: masked)
- `SSH_PRIVATE_KEY` - приватный SSH ключ для деплоя на сервер (тип: file)
- `DEPLOY_SERVER` - IP адрес или hostname сервера для деплоя (например: `192.168.1.100`)
- `DEPLOY_USER` - пользователь для SSH подключения (например: `deploy`)
- `APP_VERSION` - версия приложения (например: `1.0.0`)

**3. Включить Container Registry:**

- Проверить что Registry включен в проекте
- Проверить доступность: свой локальный регистри

**4. Настроить GitLab Runner:**

Настроить свой локальный раннер

**5. Добавить SSH ключ на целевой сервер:**

- Сгенерировать SSH ключ: `ssh-keygen -t ed25519 -C "gitlab-ci"`
- Добавить публичный ключ на сервер в `~/.ssh/authorized_keys`
- Приватный ключ добавить в GitLab переменные

Pipe1: Веб-приложение
План реализации:
1 - Написать веб приложение на flask
2 - Определить зависимости
3 - Написать тесты

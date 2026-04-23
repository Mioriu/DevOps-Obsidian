Каждая команда мечтает об идеальном процессе разработки: код всегда проходит проверки, коммиты имеют правильный формат, тесты запускаются автоматически, а в production никогда не попадает невалидный код. Git hooks превращают эту мечту в реальность, позволяя автоматизировать проверки и действия на каждом этапе работы с репозиторием. Это встроенный в Git механизм запуска скриптов при определённых событиях — от создания коммита до получения push на сервере. Правильно настроенные hooks экономят часы работы и предотвращают типичные ошибки.

### Что такое Git hooks

**Git hooks** — это скрипты, которые Git автоматически выполняет до или после определённых событий: commit, push, merge и других. Hooks позволяют встроить проверки качества кода, автоматизацию рутинных задач и соблюдение стандартов прямо в workflow разработки.

#### Где находятся hooks

```diff
# В каждом Git репозитории
$ ls -la .git/hooks/
total 96
-rwxr-xr-x  applypatch-msg.sample
-rwxr-xr-x  commit-msg.sample
-rwxr-xr-x  fsmonitor-watchman.sample
-rwxr-xr-x  post-update.sample
-rwxr-xr-x  pre-applypatch.sample
-rwxr-xr-x  pre-commit.sample
-rwxr-xr-x  pre-merge-commit.sample
-rwxr-xr-x  pre-push.sample
-rwxr-xr-x  pre-rebase.sample
-rwxr-xr-x  pre-receive.sample
-rwxr-xr-x  prepare-commit-msg.sample
-rwxr-xr-x  push-to-checkout.sample
-rwxr-xr-x  update.sample
```

Git создаёт примеры hooks с расширением .sample. Чтобы активировать hook, нужно убрать расширение и сделать файл исполняемым.

#### Типы hooks

Git hooks делятся на две категории:

- **Клиентские (client-side)** — выполняются на машине разработчика
- **Серверные (server-side)** — выполняются на Git-сервере при push

### Клиентские hooks

#### pre-commit

Выполняется перед созданием коммита. Идеален для проверки кода, форматирования и запуска быстрых тестов.

```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "Running pre-commit checks..."

# Проверка синтаксиса Python файлов
for file in $(git diff --cached --name-only --diff-filter=ACM | grep '\.py$'); do
    python -m py_compile "$file"
    if [ $? -ne 0 ]; then
        echo "Syntax error in $file"
        exit 1
    fi
done

# Проверка на console.log в JavaScript
if git diff --cached --name-only | xargs grep -E "console\.(log|debug|info|warn|error)" > /dev/null; then
    echo "❌ Found console.log statements. Please remove them."
    echo "Files with console.log:"
    git diff --cached --name-only | xargs grep -l "console\."
    exit 1
fi

# Запуск линтера
if command -v eslint > /dev/null; then
    git diff --cached --name-only --diff-filter=ACM | grep '\.js$' | xargs eslint
    if [ $? -ne 0 ]; then
        echo "❌ ESLint found issues"
        exit 1
    fi
fi

# Проверка размера файлов
for file in $(git diff --cached --name-only); do
    file_size=$(ls -l "$file" | awk '{print $5}')
    if [ "$file_size" -gt 5242880 ]; then  # 5MB
        echo "❌ File $file is larger than 5MB"
        echo "Consider using Git LFS for large files"
        exit 1
    fi
done

echo "✅ All pre-commit checks passed"
```

#### prepare-commit-msg

Позволяет изменить сообщение коммита перед открытием редактора. Полезно для добавления шаблонов или автоматической информации.

```bash
#!/bin/sh
# .git/hooks/prepare-commit-msg

COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2
SHA1=$3

# Добавляем номер задачи из имени ветки
BRANCH_NAME=$(git symbolic-ref --short HEAD)
TICKET_ID=$(echo "$BRANCH_NAME" | grep -Eo '[A-Z]+-[0-9]+')

if [ -n "$TICKET_ID" ]; then
    # Проверяем, не содержит ли уже сообщение ticket ID
    if ! grep -q "$TICKET_ID" "$COMMIT_MSG_FILE"; then
        echo "[$TICKET_ID] $(cat $COMMIT_MSG_FILE)" > "$COMMIT_MSG_FILE"
    fi
fi

# Добавляем шаблон для пустых коммитов
if [ -z "$COMMIT_SOURCE" ]; then
    cat > "$COMMIT_MSG_FILE" << EOF
# Type: feat|fix|docs|style|refactor|test|chore
# Scope: optional
# Subject: imperative mood, max 50 chars
# Body: optional, wrap at 72 chars
# Footer: optional, issues closed

EOF
fi
```

#### commit-msg

Проверяет финальное сообщение коммита. Идеально для enforcing commit message conventions.

```bash
#!/bin/sh
# .git/hooks/commit-msg

commit_regex='^(feat|fix|docs|style|refactor|test|chore)(\([a-z]+\))?: .{1,50}$'
error_msg="❌ Commit message doesn't follow conventions

Format: (): 

Types:
  feat:     New feature
  fix:      Bug fix
  docs:     Documentation changes
  style:    Code style changes
  refactor: Code refactoring
  test:     Test updates
  chore:    Build process or auxiliary tool changes

Example: feat(auth): add OAuth2 integration"

if ! grep -qE "$commit_regex" "$1"; then
    echo "$error_msg" >&2
    exit 1
fi

# Проверка длины первой строки
first_line=$(head -n1 "$1")
if [ ${#first_line} -gt 72 ]; then
    echo "❌ First line of commit message is too long (${#first_line} > 72)"
    exit 1
fi

echo "✅ Commit message validation passed"
```

#### pre-push

Выполняется перед push. Отлично подходит для запуска тестов и финальных проверок.

```bash
#!/bin/sh
# .git/hooks/pre-push

remote="$1"
url="$2"

echo "Running pre-push checks..."

# Защита от push в main/master
protected_branches="main master production"
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')

for branch in $protected_branches; do
    if [ "$current_branch" = "$branch" ]; then
        echo "❌ Direct push to $branch branch is not allowed"
        echo "Please create a pull request instead"
        exit 1
    fi
done

# Запуск тестов
if [ -f "package.json" ]; then
    echo "Running tests..."
    npm test
    if [ $? -ne 0 ]; then
        echo "❌ Tests failed. Push aborted."
        exit 1
    fi
fi

# Проверка на незакоммиченные изменения
if ! git diff-index --quiet HEAD --; then
    echo "❌ You have uncommitted changes"
    echo "Please commit or stash them before pushing"
    exit 1
fi

# Проверка на TODO и FIXME
if git diff --cached --name-only | xargs grep -E "(TODO|FIXME)" > /dev/null; then
    echo "⚠️  Warning: Found TODO/FIXME comments:"
    git diff --cached --name-only | xargs grep -n -E "(TODO|FIXME)"
    read -p "Continue with push? (y/n) " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        exit 1
    fi
fi

echo "✅ All pre-push checks passed"
```

#### post-merge

Выполняется после успешного merge. Полезен для обновления зависимостей или очистки.

```bash
#!/bin/sh
# .git/hooks/post-merge

echo "Post-merge hook running..."

# Проверка изменений в package.json
if git diff-tree -r --name-only --no-commit-id ORIG_HEAD HEAD | grep -q "package.json"; then
    echo "📦 package.json changed, running npm install..."
    npm install
fi

# Проверка изменений в requirements.txt
if git diff-tree -r --name-only --no-commit-id ORIG_HEAD HEAD | grep -q "requirements.txt"; then
    echo "🐍 requirements.txt changed, updating Python dependencies..."
    pip install -r requirements.txt
fi

# Проверка изменений в миграциях базы данных
if git diff-tree -r --name-only --no-commit-id ORIG_HEAD HEAD | grep -q "migrations/"; then
    echo "🗄️  Database migrations detected"
    echo "Run 'python manage.py migrate' to apply them"
fi

# Очистка старых файлов
echo "🧹 Cleaning up..."
find . -name "*.pyc" -delete
find . -name "__pycache__" -type d -delete

echo "✅ Post-merge tasks completed"
```

### Серверные hooks

#### pre-receive

Выполняется на сервере перед принятием push. Может отклонить весь push.

```bash
#!/bin/sh
# hooks/pre-receive

while read oldrev newrev refname; do
    echo "Checking push to $refname..."
    
    # Блокировка force push в protected branches
    if [ "$refname" = "refs/heads/main" ] || [ "$refname" = "refs/heads/master" ]; then
        # Проверка на force push
        if [ "$oldrev" != "0000000000000000000000000000000000000000" ]; then
            if ! git merge-base --is-ancestor "$oldrev" "$newrev"; then
                echo "❌ Force push to protected branch is not allowed"
                exit 1
            fi
        fi
    fi
    
    # Проверка размера коммитов
    commits=$(git rev-list "$oldrev".."$newrev")
    for commit in $commits; do
        # Проверка размера файлов в коммите
        large_files=$(git diff-tree -r --name-only --diff-filter=AMT "$commit" | \
                      xargs -I {} git cat-file -s "$commit":{} | \
                      awk '$1 > 10485760 {print}')
        
        if [ -n "$large_files" ]; then
            echo "❌ Commit $commit contains files larger than 10MB"
            echo "Please use Git LFS for large files"
            exit 1
        fi
        
        # Проверка commit message
        msg=$(git log -1 --pretty=%B "$commit")
        if ! echo "$msg" | grep -qE '^(feat|fix|docs|style|refactor|test|chore)'; then
            echo "❌ Commit $commit has invalid message format"
            echo "Message: $msg"
            exit 1
        fi
    done
done

echo "✅ All server-side checks passed"
```

#### update

Вызывается для каждой обновляемой ветки. Позволяет более гранулярный контроль.

```bash
#!/bin/sh
# hooks/update

refname="$1"
oldrev="$2"
newrev="$3"

# Проверка прав доступа
user=$(git config user.email)
branch=$(echo "$refname" | sed 's/refs\/heads\///')

# Список администраторов
admins="admin@company.com lead@company.com"

# Защита production веток
if [ "$branch" = "production" ]; then
    if ! echo "$admins" | grep -q "$user"; then
        echo "❌ Only administrators can push to production"
        echo "Your email: $user"
        exit 1
    fi
fi

# Проверка на удаление веток
if [ "$newrev" = "0000000000000000000000000000000000000000" ]; then
    if [ "$branch" = "main" ] || [ "$branch" = "develop" ]; then
        echo "❌ Deletion of $branch branch is not allowed"
        exit 1
    fi
fi

# Логирование
echo "$(date): $user updated $branch from $oldrev to $newrev" >> hooks.log
```

#### post-receive

Выполняется после успешного получения push. Идеален для развёртывания и уведомлений.

```bash
#!/bin/sh
# hooks/post-receive

while read oldrev newrev refname; do
    branch=$(echo "$refname" | sed 's/refs\/heads\///')
    
    # Автоматическое развёртывание
    if [ "$branch" = "production" ]; then
        echo "🚀 Deploying to production..."
        
        # Обновление рабочей копии
        GIT_WORK_TREE=/var/www/production git checkout -f production
        
        # Перезапуск сервисов
        systemctl restart myapp
        
        # Отправка уведомления
        curl -X POST https://hooks.slack.com/services/xxx/yyy/zzz \
             -H 'Content-type: application/json' \
             --data "{\"text\":\"Production deployed: $newrev\"}"
    fi
    
    # Запуск CI/CD
    if [ "$branch" = "develop" ]; then
        echo "🔧 Triggering CI build..."
        curl -X POST https://ci.company.com/api/builds \
             -H "Authorization: Bearer $CI_TOKEN" \
             -d "{\"branch\": \"$branch\", \"commit\": \"$newrev\"}"
    fi
    
    # Обновление документации
    if git diff-tree --name-only -r "$oldrev".."$newrev" | grep -q "^docs/"; then
        echo "📚 Rebuilding documentation..."
        cd /var/www/docs && make html
    fi
done
```

### Управление hooks в команде

Главная проблема Git hooks — они не версионируются. Каждый разработчик должен настроить их локально.

#### Использование директории hooks

```bash
# Создаём директорию для hooks в репозитории
$ mkdir .githooks

# Копируем hooks
$ cp .git/hooks/pre-commit .githooks/
$ git add .githooks/pre-commit
$ git commit -m "Add pre-commit hook"

# Скрипт установки для команды
$ cat > scripts/install-hooks.sh << 'EOF'
#!/bin/sh
echo "Installing git hooks..."

# Создаём символические ссылки
for hook in .githooks/*; do
    hook_name=$(basename "$hook")
    ln -sf "../../.githooks/$hook_name" ".git/hooks/$hook_name"
    chmod +x ".git/hooks/$hook_name"
done

echo "✅ Git hooks installed successfully"
EOF

$ chmod +x scripts/install-hooks.sh
```

#### Git config для hooks

```bash
# Git 2.9+ поддерживает кастомную директорию hooks
$ git config core.hooksPath .githooks

# Для всей команды в README
echo "Run: git config core.hooksPath .githooks" >> README.md
```

### Инструменты для управления hooks

#### Husky (JavaScript)

```ruby
# Установка
$ npm install husky --save-dev

# Инициализация
$ npx husky install

# Добавление hook
$ npx husky add .husky/pre-commit "npm test"
$ npx husky add .husky/commit-msg 'npx commitlint --edit "$1"'

# package.json
{
  "scripts": {
    "prepare": "husky install"
  }
}
```

#### pre-commit (Python)

```yaml
# Установка
$ pip install pre-commit

# Конфигурация .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=1000']
      
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
        
  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
        args: ['--max-line-length=88']

# Установка hooks
$ pre-commit install

# Запуск вручную
$ pre-commit run --all-files
```

#### Lefthook

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    eslint:
      glob: "*.{js,jsx,ts,tsx}"
      run: npx eslint {staged_files}
    prettier:
      glob: "*.{js,jsx,ts,tsx,css,scss}"
      run: npx prettier --check {staged_files}
    tests:
      run: npm test

commit-msg:
  commands:
    commitlint:
      run: npx commitlint --edit {1}

pre-push:
  commands:
    tests:
      run: npm run test:ci
    audit:
      run: npm audit

# Установка
$ npm install --save-dev lefthook
$ npx lefthook install
```

### Продвинутые техники

#### Условное выполнение hooks

```bash
#!/bin/sh
# .git/hooks/pre-commit с условиями

# Пропуск hooks при rebase
if [ -f "$(git rev-parse --git-dir)/REBASE_HEAD" ]; then
    echo "Skipping pre-commit during rebase"
    exit 0
fi

# Пропуск для определённых веток
current_branch=$(git symbolic-ref --short HEAD)
if [ "$current_branch" = "hotfix" ]; then
    echo "Skipping checks for hotfix branch"
    exit 0
fi

# Пропуск по переменной окружения
if [ "$SKIP_HOOKS" = "1" ]; then
    echo "Skipping hooks (SKIP_HOOKS=1)"
    exit 0
fi

# Обычная логика hook...
```

#### Hook с параметрами

```bash
#!/bin/sh
# .git/hooks/pre-commit с конфигурацией

# Загрузка конфигурации
if [ -f ".hookconfig" ]; then
    . ./.hookconfig
fi

# Значения по умолчанию
MAX_FILE_SIZE=${MAX_FILE_SIZE:-5242880}  # 5MB
LINT_ENABLED=${LINT_ENABLED:-true}
TEST_ENABLED=${TEST_ENABLED:-false}

# Использование конфигурации
if [ "$LINT_ENABLED" = "true" ]; then
    echo "Running linter..."
    # логика линтера
fi
```

#### Интеграция с Docker

```bash
#!/bin/sh
# .git/hooks/pre-commit для Docker окружения

# Запуск проверок внутри контейнера
docker run --rm \
    -v "$(pwd)":/app \
    -w /app \
    myproject:dev \
    sh -c "npm run lint && npm test"

if [ $? -ne 0 ]; then
    echo "❌ Checks failed inside Docker"
    exit 1
fi
```

### Отладка hooks

#### Логирование

```bash
#!/bin/sh
# Добавляем отладочную информацию

# Включение отладки
set -x  # Печатает каждую команду
set -e  # Останавливается при ошибке

# Логирование в файл
LOG_FILE=".git/hooks.log"
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

echo "=== Hook: $0 ==="
echo "Date: $(date)"
echo "User: $(git config user.name)"
echo "Args: $@"
echo "PWD: $(pwd)"
echo "---"

# Основная логика...
```

#### Тестирование hooks

```shell
# Запуск hook вручную
$ .git/hooks/pre-commit

# С отладкой
$ bash -x .git/hooks/pre-commit

# Тестовый коммит без выполнения
$ git commit --no-verify -m "Test commit"

# Эмуляция окружения Git
$ GIT_DIR=.git GIT_INDEX_FILE=.git/index .git/hooks/pre-commit
```

### Безопасность hooks

#### Валидация входных данных

```bash
#!/bin/bash
# Безопасный hook

# Валидация аргументов
if [ $# -ne 3 ]; then
    echo "Error: Invalid number of arguments"
    exit 1
fi

refname="$1"
oldrev="$2"
newrev="$3"

# Проверка формата SHA
if ! echo "$oldrev" | grep -qE '^[0-9a-f]{40}$'; then
    echo "Error: Invalid oldrev format"
    exit 1
fi

# Экранирование для использования в командах
safe_refname=$(printf '%q' "$refname")

# Использование --end-of-options
git log --end-of-options "$oldrev..$newrev"
```

#### Ограничение прав

```bash
#!/bin/sh
# Hook с ограниченными правами

# Запуск с ограниченными правами
if [ "$(id -u)" = "0" ]; then
    echo "❌ Refusing to run as root"
    exit 1
fi

# Проверка переменных окружения
required_vars="GIT_DIR GIT_OBJECT_DIRECTORY"
for var in $required_vars; do
    if [ -z "$(eval echo \$$var)" ]; then
        echo "❌ Required variable $var is not set"
        exit 1
    fi
done

# Очистка PATH
export PATH="/usr/local/bin:/usr/bin:/bin"
```

### Альтернативы Git hooks

#### CI/CD проверки

GitHub Actions, GitLab CI, Jenkins — выполняют проверки на сервере:

```yaml
# .github/workflows/checks.yml
name: Code Quality
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run linter
        run: |
          npm install
          npm run lint
```

#### IDE интеграция

Многие IDE имеют встроенные проверки:

- VS Code: Format on Save, ESLint extension
- IntelliJ IDEA: Code inspections, commit checks
- Vim: ALE, syntastic plugins

#### Git aliases с проверками

```ruby
# Добавляем проверки в Git aliases
$ git config alias.safe-commit '!f() { npm test && git commit "$@"; }; f'
$ git config alias.push-with-tests '!f() { npm test && git push "$@"; }; f'
```

### Лучшие практики

- **Быстрые hooks** — pre-commit должен работать секунды, не минуты
- **Возможность пропуска** — всегда оставляйте --no-verify для экстренных случаев
- **Информативные сообщения** — объясняйте, что пошло не так и как исправить
- **Версионирование** — храните hooks в репозитории
- **Документация** — описывайте, какие hooks используются и зачем
- **Постепенное внедрение** — начните с предупреждений, потом делайте обязательными
- **Кроссплатформенность** — учитывайте разные ОС в команде

## Итог

Git hooks — это мощный инструмент автоматизации, который позволяет встроить проверки качества прямо в процесс разработки. Правильно настроенные hooks предотвращают попадание невалидного кода в репозиторий, автоматизируют рутинные задачи и помогают поддерживать единые стандарты в команде. Ключ к успеху — баланс между автоматизацией и гибкостью: hooks должны помогать, а не мешать разработке. Начните с простых проверок, постепенно расширяйте функциональность и обязательно документируйте процессы для команды. Современные инструменты вроде Husky и pre-commit делают управление hooks простым и удобным, позволяя сосредоточиться на разработке, а не на инфраструктуре.Fcore
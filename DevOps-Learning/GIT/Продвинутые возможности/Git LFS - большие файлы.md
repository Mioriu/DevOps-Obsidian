Git прекрасно работает с исходным кодом, но что делать с большими файлами? Дизайнерские макеты, видео, 3D-модели, датасеты — всё это может весить гигабайты. Добавить их в обычный Git-репозиторий — значит сделать его неповоротливым монстром, который часами клонируется и съедает весь диск. Git Large File Storage (LFS) решает эту проблему, сохраняя большие файлы отдельно от основного репозитория, но при этом версионируя их вместе с кодом. Это элегантное решение, которое сохраняет простоту Git и добавляет эффективность работы с большими данными.

### Почему Git плохо работает с большими файлами

Чтобы понять ценность Git LFS, важно разобраться в ограничениях обычного Git.

#### Проблемы хранения

Git хранит полную копию каждой версии файла. Изменили один байт в 100-мегабайтном файле? Git сохранит обе версии целиком. После десятка изменений репозиторий распухнет до гигабайта, хотя реальные изменения минимальны.

```shell
# Пример роста репозитория
$ ls -lh design.psd
-rw-r--r-- 1 user user 100M Jan 1 10:00 design.psd

# После 10 коммитов с изменениями
$ du -sh .git
1.1G    .git  # Репозиторий вырос до 1.1 ГБ!
```

#### Проблемы производительности

- **Клонирование** — каждый clone скачивает всю историю всех файлов
- **Сборка мусора** — git gc работает медленно на больших файлах
- **Сетевые операции** — push/pull становятся болезненно долгими
- **Память** — операции требуют загрузки файлов в память целиком

#### Платформенные ограничения

GitHub, GitLab и другие платформы имеют жёсткие ограничения:

- GitHub: файлы больше 100 МБ блокируются, больше 50 МБ — предупреждение
- GitLab: лимит на размер репозитория (5-10 ГБ в бесплатной версии)
- Bitbucket: ограничение на размер репозитория 2 ГБ

### Как работает Git LFS

Git LFS заменяет большие файлы в репозитории на маленькие текстовые указатели (pointer files), а сами файлы хранит отдельно на LFS-сервере.

#### Архитектура Git LFS

```less
# Обычный Git
Repository:
├── code.js (2 KB) → хранится в Git
├── image.jpg (5 MB) → хранится в Git
└── video.mp4 (500 MB) → хранится в Git

# Git с LFS
Repository:
├── code.js (2 KB) → хранится в Git
├── image.jpg (134 bytes) → указатель в Git, файл в LFS
└── video.mp4 (134 bytes) → указатель в Git, файл в LFS
```

#### Структура указателя

```bash
$ cat video.mp4  # На самом деле это текстовый указатель
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a2145b3c3b7f8c8e9d1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2
size 524288000
```

Указатель содержит:

- **version** — версия спецификации LFS
- **oid** — SHA256 хеш содержимого файла
- **size** — размер файла в байтах

### Установка и настройка Git LFS

#### Установка

```shell
# macOS
$ brew install git-lfs

# Ubuntu/Debian
$ sudo apt-get install git-lfs

# Windows (с Git for Windows уже включён)
# Или через Chocolatey
$ choco install git-lfs

# Из исходников
$ wget https://github.com/git-lfs/git-lfs/releases/download/v3.3.0/git-lfs-linux-amd64-v3.3.0.tar.gz
$ tar -xzf git-lfs-linux-amd64-v3.3.0.tar.gz
$ sudo ./install.sh
```

#### Инициализация в системе

```erlang
# Одноразовая настройка Git LFS для пользователя
$ git lfs install
Updated git hooks.
Git LFS initialized.

# Что произошло
$ cat ~/.gitconfig
[filter "lfs"]
    clean = git-lfs clean -- %f
    smudge = git-lfs smudge -- %f
    process = git-lfs filter-process
    required = true
```

Git LFS использует фильтры Git для прозрачной работы с большими файлами.

### Базовое использование Git LFS

#### Настройка отслеживания файлов

```ruby
# Начинаем отслеживать определённые типы файлов
$ git lfs track "*.psd"
Tracking "*.psd"

$ git lfs track "*.mp4"
Tracking "*.mp4"

# Отслеживание по пути
$ git lfs track "assets/videos/*"
$ git lfs track "datasets/**/*.csv"

# Проверяем настройки
$ cat .gitattributes
*.psd filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
assets/videos/* filter=lfs diff=lfs merge=lfs -text
datasets/**/*.csv filter=lfs diff=lfs merge=lfs -text
```

Важно: файл .gitattributes должен быть закоммичен в репозиторий!

#### Добавление файлов

```shell
# Добавляем .gitattributes
$ git add .gitattributes
$ git commit -m "Configure Git LFS tracking"

# Теперь добавляем большой файл
$ cp ~/Downloads/design.psd .
$ git add design.psd
$ git commit -m "Add design mockup"

# Git LFS автоматически перехватил файл
$ git lfs ls-files
7d865e5916 * design.psd
```

### Работа с существующим репозиторием

#### Миграция существующих файлов на LFS

```ruby
# Анализируем репозиторий на большие файлы
$ git lfs migrate info --everything
migrate: Sorting commits: ..., done.
migrate: Examining commits: 100% (237/237), done.
*.mp4   1.2 GB    12/12 files(s)  100%
*.psd   800 MB     8/8 files(s)   100%
*.zip   300 MB     3/3 files(s)   100%

# Мигрируем конкретные типы файлов
$ git lfs migrate import --include="*.mp4,*.psd" --everything

# Или только определённые ветки
$ git lfs migrate import --include="*.mp4" --include-ref=main --include-ref=develop

# После миграции нужен force push
$ git push --force --all
```

Внимание: миграция переписывает историю!

#### Выборочная миграция

```haskell
# Мигрировать только файлы больше определённого размера
$ git lfs migrate import --above=50MB

# Исключить определённые пути
$ git lfs migrate import --include="*.jpg" --exclude="thumbnails/*"

# Проверка без изменений (dry run)
$ git lfs migrate import --include="*.psd" --dry-run
```

### Операции с LFS-файлами

#### Просмотр LFS-файлов

```ruby
# Список всех LFS-файлов в репозитории
$ git lfs ls-files
8d3c4b2a15 * assets/video.mp4
7d865e5916 * design.psd

# С размерами
$ git lfs ls-files -s
assets/video.mp4 (523 MB)
design.psd (156 MB)

# Проверить, какие файлы отслеживаются
$ git lfs track
Listing tracked patterns
    *.psd (.gitattributes)
    *.mp4 (.gitattributes)

# Информация о конкретном файле
$ git lfs pointer --check design.psd
Git LFS pointer for design.psd
```

#### Управление загрузкой файлов

```ruby
# Загрузить все LFS-файлы для текущего коммита
$ git lfs pull

# Загрузить файлы с удалённого репозитория
$ git lfs pull origin main

# Загрузить только определённые файлы
$ git lfs pull --include="*.mp4"
$ git lfs pull --exclude="datasets/*"

# Загрузить файлы для определённого коммита
$ git lfs fetch --all
$ git lfs checkout
```

### Оптимизация работы с LFS

#### Частичное клонирование

```shell
# Клонировать без автоматической загрузки LFS-файлов
$ GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/user/repo.git

# Или настроить глобально
$ git config --global filter.lfs.smudge "git-lfs smudge --skip -- %f"
$ git config --global filter.lfs.process "git-lfs filter-process --skip"

# Загрузить только нужные файлы
$ cd repo
$ git lfs pull --include="src/assets/logo.psd"
```

#### Работа с указателями вместо файлов

```ruby
# Конвертировать LFS-файл обратно в указатель
$ git lfs pointer --file=video.mp4 > video.mp4

# Проверить, является ли файл указателем
$ git lfs pointer --check --file=video.mp4

# Сравнить указатель с файлом
$ git lfs pointer --file=video.mp4 | git hash-object --stdin
```

### Настройка сервера и хранилища

#### Использование custom LFS-сервера

```shell
# Настроить URL LFS-сервера для репозитория
$ git config lfs.url https://lfs.company.com/repo/lfs

# Или в .lfsconfig (коммитится в репозиторий)
$ cat .lfsconfig
[lfs]
    url = https://lfs.company.com/repo/lfs

# Проверить настройки
$ git lfs env
git-lfs/3.3.0
git version 2.39.0
Endpoint=https://lfs.company.com/repo/lfs (auth=basic)
```

#### Аутентификация

```shell
# Basic auth
$ git config lfs.https://lfs.company.com.access basic

# Использование credential helper
$ git config credential.helper store

# SSH для LFS
$ git config lfs.ssh://git@github.com:user/repo.git.sshtransfer true
```

### Продвинутые возможности

#### Блокировка файлов

Git LFS поддерживает блокировку файлов для предотвращения конфликтов при работе с бинарными файлами:

```shell
# Включить поддержку блокировок
$ git lfs track "*.psd" --lockable

# Заблокировать файл
$ git lfs lock design.psd
Locked design.psd

# Посмотреть заблокированные файлы
$ git lfs locks
design.psd  johndoe  ID:12345

# Разблокировать
$ git lfs unlock design.psd

# Принудительная разблокировка (для админов)
$ git lfs unlock --force design.psd --id=12345
```

#### Настройка производительности

```ruby
# Увеличить количество параллельных передач
$ git config lfs.concurrenttransfers 8

# Настроить размер батча
$ git config lfs.batch true
$ git config lfs.transfer.maxretries 3

# Прогресс загрузки
$ git config lfs.transfer.enablehashing false  # Отключить для ускорения
```

#### Pruning и очистка

```ruby
# Удалить локальные LFS-файлы, не используемые в последних коммитах
$ git lfs prune

# Проверка без удаления
$ git lfs prune --dry-run

# Сохранить файлы за последние N дней
$ git lfs prune --days=7

# Проверить, какие файлы будут удалены
$ git lfs prune --verify-remote
```

### Решение типичных проблем

#### Файл добавлен в Git вместо LFS

```ruby
# Проблема: большой файл попал в обычный Git
$ git rm --cached large-file.zip
$ git add large-file.zip
$ git commit --amend

# Если уже запушено
$ git lfs migrate import --include="large-file.zip" --everything
$ git push --force
```

#### Ошибки при push

```shell
# Ошибка: LFS upload failed
$ git push
LFS upload failed:
  (missing) large-file.zip (oid: abc123...)

# Решение 1: Загрузить файл
$ git lfs push --all origin

# Решение 2: Проверить наличие файла
$ git lfs fsck

# Решение 3: Пересоздать указатель
$ git lfs pointer --file=large-file.zip > temp
$ mv temp large-file.zip
$ git add large-file.zip
```

#### Проблемы с местом на диске

```shell
# Очистить локальный кеш LFS
$ git lfs prune

# Удалить все локальные LFS-объекты
$ rm -rf .git/lfs/objects

# Загрузить только для текущей ветки
$ git lfs fetch origin --recent
```

### LFS в CI/CD

#### GitHub Actions

```yaml
name: Build with LFS
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          lfs: true
      
      # Или выборочная загрузка
      - name: Checkout without LFS
        uses: actions/checkout@v3
        
      - name: Pull specific LFS files
        run: |
          git lfs pull --include="assets/icons/*"
```

#### GitLab CI

```yaml
build:
  script:
    - git lfs pull --include="*.jpg" --exclude="raw/*"
    - make build
  
  # Или глобально
  variables:
    GIT_STRATEGY: clone
    GIT_LFS_SKIP_SMUDGE: 1
```

#### Jenkins

```less
pipeline {
    agent any
    
    options {
        skipDefaultCheckout()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [
                        [$class: 'GitLFSPull',
                         include: '*.png,*.jpg']
                    ]
                ])
            }
        }
    }
}
```

### Альтернативы Git LFS

#### Git Annex

Более гибкий, но сложный инструмент:

```shell
# Инициализация
$ git annex init

# Добавление файла
$ git annex add large-file.zip
$ git commit -m "Add large file"

# Синхронизация
$ git annex sync --content
```

#### DVC (Data Version Control)

Специализирован для ML/DS проектов:

```shell
# Инициализация
$ dvc init

# Добавление файла
$ dvc add data/dataset.csv
$ git add data/dataset.csv.dvc
$ git commit -m "Add dataset"

# Настройка хранилища
$ dvc remote add -d myremote s3://mybucket/path
```

#### Облачные хранилища с версионированием

- AWS S3 с версионированием
- Google Cloud Storage
- Azure Blob Storage

### Лучшие практики Git LFS

#### Что хранить в LFS

**Подходит для LFS:**

- Бинарные файлы, которые не сжимаются (изображения, видео, аудио)
- Скомпилированные файлы (если нужно версионировать)
- Большие датасеты
- Архивы и дистрибутивы

**НЕ подходит для LFS:**

- Исходный код (даже большие файлы)
- Файлы, которые часто изменяются
- Маленькие бинарные файлы (< 1 МБ)
- Временные или генерируемые файлы

#### Оптимальная настройка .gitattributes

```python
# Изображения
*.jpg filter=lfs diff=lfs merge=lfs -text
*.jpeg filter=lfs diff=lfs merge=lfs -text
*.png filter=lfs diff=lfs merge=lfs -text
*.gif filter=lfs diff=lfs merge=lfs -text
*.psd filter=lfs diff=lfs merge=lfs -text
*.ai filter=lfs diff=lfs merge=lfs -text
*.sketch filter=lfs diff=lfs merge=lfs -text

# Видео
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.mov filter=lfs diff=lfs merge=lfs -text
*.avi filter=lfs diff=lfs merge=lfs -text
*.webm filter=lfs diff=lfs merge=lfs -text

# Аудио
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.wav filter=lfs diff=lfs merge=lfs -text
*.flac filter=lfs diff=lfs merge=lfs -text

# Архивы
*.zip filter=lfs diff=lfs merge=lfs -text
*.tar.gz filter=lfs diff=lfs merge=lfs -text
*.rar filter=lfs diff=lfs merge=lfs -text

# 3D модели
*.fbx filter=lfs diff=lfs merge=lfs -text
*.obj filter=lfs diff=lfs merge=lfs -text
*.max filter=lfs diff=lfs merge=lfs -text
*.blend filter=lfs diff=lfs merge=lfs -text

# Документы (выборочно)
*.pdf filter=lfs diff=lfs merge=lfs -text
```

#### Мониторинг использования

```shell
# Проверить размер LFS-хранилища
$ git count-objects -v -H
size-lfs: 2.3 GB

# Статистика по файлам
$ git lfs ls-files -s | awk '{sum+=$4} END {print sum/1024/1024 " MB"}'

# Найти большие файлы не в LFS
$ find . -type f -size +50M | grep -v .git | xargs ls -lh
```

### Миграция с LFS

Если нужно отказаться от LFS:

```shell
# Экспорт из LFS обратно в Git
$ git lfs migrate export --include="*.jpg" --everything

# Удалить LFS из репозитория
$ git lfs uninstall

# Очистить конфигурацию
$ git lfs untrack "*.jpg"
$ rm .gitattributes
$ git add .gitattributes
$ git commit -m "Remove LFS tracking"
```

## Итог

Git LFS — это мощное решение для версионирования больших файлов, которое сохраняет простоту Git и добавляет эффективность работы с большими данными. Ключевое преимущество — прозрачность: после настройки вы работаете с большими файлами как с обычными, а Git LFS заботится об оптимальном хранении и передаче. Важно правильно выбирать, какие файлы хранить в LFS: это должны быть большие, редко изменяемые бинарные файлы. При правильном использовании Git LFS решает проблему раздувания репозиториев, ускоряет клонирование и позволяет эффективно версионировать ресурсы проекта любого размера. Помните о необходимости настройки LFS на всех рабочих местах и в CI/CD, а также о дополнительных требованиях к серверной инфраструктуре для хранения LFS-объектов.
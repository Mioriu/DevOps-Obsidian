В разгар работы над задачей приходит срочный запрос: нужно переключиться на другую ветку и исправить критический баг. Но текущие изменения ещё не готовы для коммита — код не дописан, тесты не проходят. Создавать WIP-коммит не хочется, откатывать изменения — тем более. Git stash решает эту проблему, предоставляя временное хранилище для незавершённой работы. Это как кнопка паузы для вашего кода.

### Что такое stash и как он работает

**Git stash** — это механизм сохранения текущих изменений в специальном временном хранилище без создания коммита. Stash сохраняет состояние рабочей директории и индекса, позволяя вернуться к чистому состоянию последнего коммита, а затем восстановить сохранённые изменения.

#### Что сохраняет stash

По умолчанию stash сохраняет:

- Изменения в отслеживаемых файлах (modified files)
- Изменения в индексе (staged changes)

Не сохраняются без дополнительных опций:

- Неотслеживаемые файлы (untracked files)
- Игнорируемые файлы (ignored files)

#### Внутреннее устройство stash

Технически stash — это специальные коммиты, которые Git хранит в отдельном пространстве имён. Когда вы создаёте stash, Git создаёт два или три коммита:

1. **Коммит индекса** — сохраняет состояние staging area
2. **Коммит рабочего дерева** — сохраняет все изменения
3. **Коммит неотслеживаемых файлов** — если использована опция `-u`

```applescript
# Структура stash в Git
         .----W (рабочее дерево)
        /    /
       /    I (индекс)
      /    /
HEAD------U (неотслеживаемые файлы, опционально)
```

### Базовое использование stash

Рассмотрим основные операции со stash на практических примерах.

#### Сохранение изменений

```bash
# Проверяем текущее состояние
$ git status
On branch feature/new-design
Changes to be committed:
  modified:   src/components/Header.js
Changes not staged for commit:
  modified:   src/components/Footer.js
  modified:   src/styles/main.css
Untracked files:
  src/components/Test.js

# Сохраняем изменения в stash
$ git stash
Saved working directory and index state WIP on feature/new-design: a3b4c5d Update layout

# Рабочая директория теперь чистая
$ git status
On branch feature/new-design
nothing to commit, working tree clean
```

Git автоматически создаёт описание stash на основе текущей ветки и последнего коммита.

#### Сохранение с описанием

```ruby
# Добавить понятное описание
$ git stash save "Work in progress: responsive navigation"

# Или в новом синтаксисе
$ git stash push -m "Work in progress: responsive navigation"
```

Описательные сообщения критически важны, когда у вас несколько stash-записей.

#### Восстановление изменений

Существует два основных способа восстановить изменения из stash:

```ruby
# Применить последний stash и удалить его из списка
$ git stash pop

# Применить последний stash, но оставить его в списке
$ git stash apply
```

Разница важна: `pop` = `apply` + `drop`. Используйте `apply`, если не уверены, что stash больше не понадобится.

### Работа со списком stash

Git позволяет хранить множество stash-записей одновременно, организованных в стек (последний сохранённый — первый в списке).

#### Просмотр списка

```graphql
# Показать все сохранённые stash
$ git stash list
stash@{0}: On feature/auth: Work in progress: login form
stash@{1}: WIP on main: 5a3b4c1 Fix header bug
stash@{2}: On feature/api: API client refactoring
```

Записи нумеруются от 0 (самая свежая) и имеют формат `stash@{n}`.

#### Работа с конкретным stash

```ruby
# Посмотреть содержимое stash
$ git stash show stash@{1}
 src/header.js | 15 +++++++--------
 src/styles.css | 8 ++++++--
 2 files changed, 13 insertions(+), 10 deletions(-)

# Детальный просмотр с diff
$ git stash show -p stash@{1}

# Применить конкретный stash
$ git stash apply stash@{2}

# Удалить конкретный stash
$ git stash drop stash@{1}

# Очистить все stash-записи
$ git stash clear
```

### Продвинутые возможности stash

Stash предоставляет гибкие возможности для различных сценариев.

#### Частичный stash

Можно сохранить только часть изменений:

```perl
# Интерактивный выбор изменений для stash
$ git stash push -p

# Git покажет каждое изменение и спросит
# y - добавить в stash
# n - оставить в рабочей директории
# q - завершить
# a - добавить все оставшиеся в файле
# d - пропустить все оставшиеся в файле
```

Это полезно, когда часть изменений готова к коммиту, а часть нужно отложить.

#### Сохранение неотслеживаемых файлов

```ruby
# Включить untracked файлы в stash
$ git stash push -u
# или
$ git stash push --include-untracked

# Включить даже игнорируемые файлы
$ git stash push -a
# или
$ git stash push --all
```

Опция `-u` часто необходима при переключении между ветками с разной структурой файлов.

#### Stash конкретных файлов

```shell
# Сохранить только указанные файлы
$ git stash push src/components/Header.js src/styles/header.css

# С сообщением
$ git stash push -m "Header WIP" src/components/Header.js

# Сохранить всё, кроме указанных файлов
$ git stash push --keep-index
```

### Создание ветки из stash

Иногда временные изменения перерастают в полноценную задачу:

```1c
# Создать новую ветку и применить stash
$ git stash branch feature/experiment stash@{1}

# Что происходит:
# 1. Создаётся ветка от коммита, где был создан stash
# 2. Применяются изменения из stash
# 3. Stash удаляется из списка (если применился без конфликтов)
```

Это гарантирует, что stash применится без конфликтов, так как ветка создаётся от того же состояния, где stash был сохранён.

### Типичные сценарии использования

#### Срочное переключение контекста

```shell
# Работаем над новой функцией
$ git status
Changes not staged for commit:
  modified:   src/feature.js

# Приходит срочный баг
$ git stash push -m "Feature work in progress"
$ git checkout main
$ git pull origin main

# Исправляем баг
$ git checkout -b hotfix/critical-bug
# ... работа над исправлением ...
$ git commit -m "Fix critical bug"
$ git push origin hotfix/critical-bug

# Возвращаемся к работе
$ git checkout feature/new-functionality
$ git stash pop
```

#### Перенос изменений между ветками

```ruby
# Начали работу не в той ветке
$ git stash
$ git checkout correct-branch
$ git stash pop
```

#### Эксперименты без коммитов

```ruby
# Сохраняем текущее состояние
$ git stash push -m "Before experiment"

# Экспериментируем
# ... изменения ...

# Если эксперимент неудачный
$ git checkout .
$ git stash pop

# Если удачный - коммитим и удаляем stash
$ git add .
$ git commit -m "Successful experiment"
$ git stash drop
```

### Конфликты при применении stash

Как и при merge, применение stash может вызвать конфликты, если файлы изменились с момента создания stash.

#### Разрешение конфликтов

```shell
$ git stash pop
Auto-merging src/components/Header.js
CONFLICT (content): Merge conflict in src/components/Header.js

# Stash не удаляется при конфликте
$ git stash list
stash@{0}: On feature: Header changes

# Разрешаем конфликт вручную
$ vim src/components/Header.js
$ git add src/components/Header.js

# Stash остался в списке, удаляем вручную
$ git stash drop
```

#### Предварительная проверка

```1c
# Проверить, будут ли конфликты
$ git stash show -p | git apply --check

# Если команда завершается без ошибок, конфликтов не будет
```

### Управление множественными stash

При активной работе может накопиться много stash-записей. Важно поддерживать порядок.

#### Именование и организация

```perl
# Используйте префиксы для группировки
$ git stash push -m "[WIP] Feature: user authentication"
$ git stash push -m "[EXP] Try new animation library"
$ git stash push -m "[BUG] Debugging production issue"

# Поиск по описанию
$ git stash list | grep WIP
```

#### Очистка старых stash

```perl
# Посмотреть дату создания stash
$ git stash list --date=relative
stash@{0}: On main: WIP (2 hours ago)
stash@{1}: On feature: Experiment (3 days ago)
stash@{2}: On develop: Old changes (2 weeks ago)

# Применить и проверить старые stash
$ git stash show stash@{2}
# Если не нужен
$ git stash drop stash@{2}
```

### Альтернативы stash

В некоторых случаях есть альтернативные подходы:

- **WIP коммит** — создать временный коммит и потом исправить через `reset` или `amend`
- **Отдельная ветка** — для долгосрочных экспериментов
- **Worktree** — работа в нескольких рабочих директориях одновременно
- **Patch файлы** — для переноса изменений между репозиториями

### Восстановление потерянного stash

Если случайно удалили нужный stash:

```perl
# Найти все stash-коммиты, включая удалённые
$ git fsck --unreachable | grep commit | cut -d' ' -f3 | 
  xargs git log --merges --no-walk --grep=WIP

# Или через reflog
$ git reflog show stash

# Восстановить конкретный коммит как stash
$ git stash store -m "Recovered stash" abc123def
```

### Практические советы

- **Всегда добавляйте описание** — через неделю "WIP on main" ничего не скажет
- **Регулярно чистите список** — старые stash редко бывают актуальны
- **Используйте `apply` вместо `pop`** — безопаснее при экспериментах
- **Проверяйте содержимое перед применением** — `git stash show -p`
- **Создавайте ветки для долгой работы** — stash для краткосрочного хранения

## Итог

Git stash — это швейцарский нож для управления незавершённой работой. Он позволяет мгновенно переключаться между задачами, экспериментировать без создания коммитов и переносить изменения между ветками. Ключевое понимание: stash — это временное решение, не замена нормальному workflow с коммитами и ветками. Используйте stash для краткосрочного хранения, давайте понятные описания и регулярно очищайте список. При правильном использовании stash делает работу с Git более гибкой и удобной, позволяя сохранять контекст работы при любых прерываниях.